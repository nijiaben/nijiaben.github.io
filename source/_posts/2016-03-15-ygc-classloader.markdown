---
layout: post
title: "JVM源码分析之自定义类加载器如何拉长YGC"
date: 2016-03-15 13:51:58 +0800
comments: true
categories: JVM GC ClassLoader
---

##概述
本文重点讲述毕玄大师在其公众号上发的一个GC问题[一个jstack/jmap等不能用的case](http://hellojava.info/?p=438)（PS：话说毕大师超级喜欢在题目里用case这个词，我觉得题目还是能尽量做到顾名思义好，不然要找起相关文章来真的好难找），对于毕大师那篇文章，题目上没有提到GC的那个问题，不过进入到文章里可以看到，既然文章提到了jstack/jmap的问题，这里也简单回答下jstack/jmap无法使用的问题，其实最常见的场景是使用jstack/jmap的用户和目标进程不是同一个用户，哪怕你执行jstack/jmap的动作是root用户也无济于事，详情可以参考我的这篇文章，[JVM Attach机制实现](http://lovestblog.cn/blog/2014/06/18/jvm-attach/),主要是讲JVM Attach机制的，不过毕大师这里主要提到的是jmap -heap/histo这两个参数带来的问题，如果使用-heap/histo的参数，其实和大家使用-F参数是一样的，底层都是通过serviceability agent来实现的，并不是jvm attach的方式，通过sa连上去之后会挂起进程，在serviceability agent里存在bug可能导致detach的动作不会被执行，从而会让进程一直挂着，可以通过top命令验证进程是否处于T状态，如果是说明进程被挂起了，如果进程被挂起了，可以通过kill -CONT [pid]来恢复。

<!--more-->

再回到那个GC的问题，用的参数如下：

```
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xms512m -Xmx512m -Xmn100m -XX:+UseConcMarkSweepGC
```

demo程序如下：

```
import com.thoughtworks.xstream.XStream;
 
public class XStreamTest {
     
    public static void main(String[] args) throws Exception {
        while(true){
            XStream xs = new XStream();
            xs.toString();
            xs = null;
        }
    }
 
}
```

执行效果如下

```
2016-03-14T22:48:01.502+0800: [GC [ParNew: 327680K->4258K(368640K), 0.0179071 secs] 327680K->4258K(1007616K), 0.0179448 secs] [Times: user=0.06 sys=0.01, real=0.01 secs] 
2016-03-14T22:48:05.975+0800: [GC [ParNew: 331938K->10239K(368640K), 0.0336279 secs] 331938K->10239K(1007616K), 0.0336593 secs] [Times: user=0.13 sys=0.02, real=0.03 secs] 
2016-03-14T22:48:12.215+0800: [GC [ParNew: 337919K->14444K(368640K), 0.0471005 secs] 337919K->14444K(1007616K), 0.0471257 secs] [Times: user=0.19 sys=0.02, real=0.05 secs] 
2016-03-14T22:48:21.768+0800: [GC [ParNew: 342124K->19088K(368640K), 0.0605017 secs] 342124K->19088K(1007616K), 0.0605295 secs] [Times: user=0.26 sys=0.03, real=0.06 secs] 
2016-03-14T22:48:35.180+0800: [GC [ParNew: 346768K->20633K(368640K), 0.0993470 secs] 346768K->25248K(1007616K), 0.0993777 secs] [Times: user=0.34 sys=0.04, real=0.09 secs]
```

发现gc的时间越来越长，但是gc触发的时机以及回收的效果都差不多，那问题究竟在哪里呢？

##Demo分析
虽然这个demo代码逻辑很简单，但是其实这是一个特殊的demo，并不简单，如果我们将XStream对象换成Object对象，会发现不存在这个问题，既然如此那有必要进去看看这个XStream的构造函数：

```
 public XStream() {
        this((ReflectionProvider)null, (Mapper)((Mapper)null), (HierarchicalStreamDriver)(new XppDriver()));
    }

    /** @deprecated */
    public XStream(ReflectionProvider reflectionProvider, Mapper mapper, HierarchicalStreamDriver driver) {
        this(reflectionProvider, driver, (ClassLoader)(new CompositeClassLoader()), mapper);
    }

    /** @deprecated */
    public XStream(ReflectionProvider reflectionProvider, HierarchicalStreamDriver driver, ClassLoader classLoader, Mapper mapper) {
        this(reflectionProvider, driver, new ClassLoaderReference(classLoader), mapper, new DefaultConverterLookup());
    }

    public XStream(ReflectionProvider reflectionProvider, HierarchicalStreamDriver driver, ClassLoaderReference classLoader, Mapper mapper, final DefaultConverterLookup defaultConverterLookup) {
        this(reflectionProvider, driver, (ClassLoaderReference)classLoader, mapper, new ConverterLookup() {
            public Converter lookupConverterForType(Class type) {
                return defaultConverterLookup.lookupConverterForType(type);
            }
        }, new ConverterRegistry() {
            public void registerConverter(Converter converter, int priority) {
                defaultConverterLookup.registerConverter(converter, priority);
            }
        });
    }

    /** @deprecated */
    public XStream(ReflectionProvider reflectionProvider, HierarchicalStreamDriver driver, ClassLoader classLoader, Mapper mapper, ConverterLookup converterLookup, ConverterRegistry converterRegistry) {
        this(reflectionProvider, driver, (ClassLoaderReference)(new ClassLoaderReference(classLoader)), mapper, converterLookup, converterRegistry);
    }

    public XStream(ReflectionProvider reflectionProvider, HierarchicalStreamDriver driver, ClassLoaderReference classLoaderReference, Mapper mapper, ConverterLookup converterLookup, ConverterRegistry converterRegistry) {
        if(reflectionProvider == null) {
            reflectionProvider = JVM.newReflectionProvider();
        }

        this.reflectionProvider = reflectionProvider;
        this.hierarchicalStreamDriver = driver;
        this.classLoaderReference = classLoaderReference;
        this.converterLookup = converterLookup;
        this.converterRegistry = converterRegistry;
        this.mapper = mapper == null?this.buildMapper():mapper;
        this.setupMappers();
        this.setupSecurity();
        this.setupAliases();
        this.setupDefaultImplementations();
        this.setupConverters();
        this.setupImmutableTypes();
        this.setMode(1003);
    }
```

这个构造函数还是很复杂的，里面会创建很多的对象，上面还有一些方法实现我就不贴了，总之都是在不断构建各种大大小小的对象，一个XStream对象构建出来的时候大概好像有12M的样子。

那到底是哪些对象会导致ygc不断增长呢，于是可能想到逐步替换上面这些逻辑，比如将最后一个构造函数里的那些逻辑都禁掉，然后我们再跑测试看看还会不会让ygc不断恶化，最终我们会发现，如果我们直接使用如下构造函数构造对象时，如果传入的classloader是AppClassLoader，那会发现这个问题不再出现了。

```
 public XStream(ReflectionProvider reflectionProvider, HierarchicalStreamDriver driver, ClassLoader classLoader, Mapper mapper) {
        this(reflectionProvider, driver, new ClassLoaderReference(classLoader), mapper, new DefaultConverterLookup());
 }
```

测试代码如下：

```      
 public static void main(String[] args) throws Exception {
        int i=0;
        while (true) {
            XStream xs = new XStream(null,null, new ClassLoaderReference(XStreamTest.class.getClassLoader()),null, new DefaultConverterLookup());
            xs.toString();
            xs=null;
        }
  }
```

gc日志如下：

```
2016-03-14T23:10:33.537+0800: [GC [ParNew: 327680K->758K(368640K), 0.0019803 secs] 327680K->758K(1007616K), 0.0020182 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2016-03-14T23:10:35.189+0800: [GC [ParNew: 328438K->1066K(368640K), 0.0018641 secs] 328438K->1066K(1007616K), 0.0019055 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
2016-03-14T23:10:36.465+0800: [GC [ParNew: 328746K->1156K(368640K), 0.0010304 secs] 328746K->1156K(1007616K), 0.0010519 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2016-03-14T23:10:37.767+0800: [GC [ParNew: 328836K->1065K(368640K), 0.0011329 secs] 328836K->1065K(1007616K), 0.0011543 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2016-03-14T23:10:39.035+0800: [GC [ParNew: 328745K->351K(368640K), 0.0043387 secs] 328745K->1127K(1007616K), 0.0043700 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
2016-03-14T23:10:40.324+0800: [GC [ParNew: 328031K->160K(368640K), 0.0011579 secs] 328807K->936K(1007616K), 0.0011793 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2016-03-14T23:10:41.610+0800: [GC [ParNew: 327840K->31K(368640K), 0.0007010 secs] 328616K->826K(1007616K), 0.0007219 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2016-03-14T23:10:42.919+0800: [GC [ParNew: 327711K->24K(368640K), 0.0011246 secs] 328506K->819K(1007616K), 0.0011450 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
2016-03-14T23:10:44.196+0800: [GC [ParNew: 327704K->24K(368640K), 0.0006797 secs] 328499K->819K(1007616K), 0.0007586 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

是不是觉得很神奇，由此可见，这个classloader至关重要。

##不得不说的类加载器
这里着重要说的两个概念是`初始类加载器`和`定义类加载器`。举个栗子说吧，AClassLoader->BClassLoader->CClassLoader，表示AClassLoader在加载类的时候会委托BClassLoader类加载器来加载，BClassLoader加载类的时候会委托CClassLoader来加载，假如我们使用AClassLoader来加载X这个类，而X这个类最终是被CClassLoader来加载的，那么我们称CClassLoader为X类的定义类加载器，而AClassLoader为X类的初始类加载器，JVM在加载某个类的时候对AClassLoader和CClassLoader进行记录，记录的数据结构是一个叫做SystemDictionary的hashtable，其key是根据ClassLoader对象和类名算出来的hash值（其实是一个entry，可以根据这个hash值找到具体的index位置，然后构建一个包含kalssName和classloader对象的entry放到map里），而value是真正的由定义类加载器加载的Klass对象，因为初始类加载器和定义类加载器是不同的classloader，因此算出来的hash值也是不同的，因此在SystemDictionary里会有多项值的value都是指向同一个Klass对象。

那么JVM为什么要分这两种类加载器呢，其实主要是为了快速找到已经加载的类，比如我们已经通过AClassLoader来触发了对X类的加载，当我们再次使用AClassLoader这个类加载器来加载X这个类的时候就不需要再委托给BClassLoader去找了，因为加载过的类在JVM里有这个类加载器的直接加载的记录，只需要直接返回对应的Klass对象即可。

##Demo中的类加载器是否会加载类
我们的demo里发现构建了一个CompositeClassLoader的类加载器，那到底有没有用这个类加载器加载类呢，我们可以设置一个断点在CompositeClassLoader的loadClass方法上，于是看到下面的堆栈：

```
main@1, prio=5, in group 'main', status: 'RUNNING'
	  at com.thoughtworks.xstream.core.util.CompositeClassLoader.loadClass(CompositeClassLoader.java:53)
	  at java.lang.Class.forName0(Class.java:-1)
	  at java.lang.Class.forName(Class.java:249)
	  at com.thoughtworks.xstream.XStream.buildMapperDynamically(XStream.java:191)
	  at com.thoughtworks.xstream.XStream.buildMapper(XStream.java:170)
	  at com.thoughtworks.xstream.XStream.<init>(XStream.java:142)
	  at com.thoughtworks.xstream.XStream.<init>(XStream.java:116)
	  at com.BBBB.main(BBBB.java:15)
```

可见确实有类加载的动作，根据类加载委托机制，在这个demo中我们能肯定类是交给AppClassLoader来加载的，这样一来CompositeClassLoader就变成了初始类加载器，而AppClassLoader会是定义类加载器，都会在SystemDictionary里存在，因此当我们不断new XStream的时候会不断new CompositeClassLoader对象，加载类的时候会不断往SystemDictionary里插入记录，从而使SystemDictionary越来越膨胀，那自然而然会想到如果GC过程不断去扫描这个SystemDictionary的话，那随着SystemDictionary不断膨胀，那么GC的效率也就越低，抱着验证下猜想的方式我们可以使用perf工具来看看，如果发现cpu占比排前的函数如果都是操作SystemDictionary的，那就基本验证了我们的说法，下面是perf工具的截图，基本证实了这一点。

{% img /images/2016/03/ygc_classloader_perf.png %}


##SystemDictionary为什么会影响GC过程
想象一下这么个情况，我们加载了一个类，然后构建了一个对象(这个对象在eden里构建)当一个属性设置到这个类里，如果gc发生的时候，这个对象是不是要被找出来标活才行，那么自然而然我们加载的类肯定是我们一项重要的gc root，这样SystemDictionary就成为了gc过程中的被扫描对象了，事实也是如此，可以看vm的具体代码：

```
void SharedHeap::process_strong_roots(bool activate_scope,
                                      bool collecting_perm_gen,
                                      ScanningOption so,
                                      OopClosure* roots,
                                      CodeBlobClosure* code_roots,
                                      OopsInGenClosure* perm_blk) {
  StrongRootsScope srs(this, activate_scope);
  // General strong roots.
  assert(_strong_roots_parity != 0, "must have called prologue code");
  // _n_termination for _process_strong_tasks should be set up stream
  // in a method not running in a GC worker.  Otherwise the GC worker
  // could be trying to change the termination condition while the task
  // is executing in another GC worker.
  if (!_process_strong_tasks->is_task_claimed(SH_PS_Universe_oops_do)) {
    Universe::oops_do(roots);
    // Consider perm-gen discovered lists to be strong.
    //将perm gen的非强引用标记为root的一部分
    perm_gen()->ref_processor()->weak_oops_do(roots);
  }
  // Global (strong) JNI handles
  if (!_process_strong_tasks->is_task_claimed(SH_PS_JNIHandles_oops_do))
    JNIHandles::oops_do(roots);
  // All threads execute this; the individual threads are task groups.
  if (ParallelGCThreads > 0) {
    Threads::possibly_parallel_oops_do(roots, code_roots);
  } else {
    Threads::oops_do(roots, code_roots);
  }
  if (!_process_strong_tasks-> is_task_claimed(SH_PS_ObjectSynchronizer_oops_do))
    ObjectSynchronizer::oops_do(roots);
  if (!_process_strong_tasks->is_task_claimed(SH_PS_FlatProfiler_oops_do))
    FlatProfiler::oops_do(roots);
  if (!_process_strong_tasks->is_task_claimed(SH_PS_Management_oops_do))
    Management::oops_do(roots);
  if (!_process_strong_tasks->is_task_claimed(SH_PS_jvmti_oops_do))
    JvmtiExport::oops_do(roots);

  if (!_process_strong_tasks->is_task_claimed(SH_PS_SystemDictionary_oops_do)) {
    if (so & SO_AllClasses) {
      SystemDictionary::oops_do(roots);
    } else if (so & SO_SystemClasses) {
      SystemDictionary::always_strong_oops_do(roots);
    }
  }

  if (!_process_strong_tasks->is_task_claimed(SH_PS_StringTable_oops_do)) {
    //JavaObjectsInPerm为false，那么String intern的对象已经class对象都是存在heap里的，否则都存在perm里  
    if (so & SO_Strings || (!collecting_perm_gen && !JavaObjectsInPerm)) {
      //虽然不回收perm，但是interned的String对象不在perm里，那么还是需要遍历下StringTable里的String对象，因为这些对象在heap里
      StringTable::oops_do(roots);
    }
    if (JavaObjectsInPerm) {
      // Verify the string table contents are in the perm gen
      NOT_PRODUCT(StringTable::oops_do(&assert_is_perm_closure));
    }
  }

  if (!_process_strong_tasks->is_task_claimed(SH_PS_CodeCache_oops_do)) {
    if (so & SO_CodeCache) {
      // (Currently, CMSCollector uses this to do intermediate-strength collections.)
      assert(collecting_perm_gen, "scanning all of code cache");
      assert(code_roots != NULL, "must supply closure for code cache");
      if (code_roots != NULL) {
        CodeCache::blobs_do(code_roots);
      }
    } else if (so & (SO_SystemClasses|SO_AllClasses)) {
      if (!collecting_perm_gen) {
        // If we are collecting from class statics, but we are not going to
        // visit all of the CodeCache, collect from the non-perm roots if any.
        // This makes the code cache function temporarily as a source of strong
        // roots for oops, until the next major collection.
        //
        // If collecting_perm_gen is true, we require that this phase will call
        // CodeCache::do_unloading.  This will kill off nmethods with expired
        // weak references, such as stale invokedynamic targets.
        CodeCache::scavenge_root_nmethods_do(code_roots);
      }
    }
    // Verify that the code cache contents are not subject to
    // movement by a scavenging collection.
    DEBUG_ONLY(CodeBlobToOopClosure assert_code_is_non_scavengable(&assert_is_non_scavengable_closure, /*do_marking=*/ false));
    DEBUG_ONLY(CodeCache::asserted_non_scavengable_nmethods_do(&assert_code_is_non_scavengable));
  }

  if (!collecting_perm_gen) {
    //如果是不回收perm，那找出所有perm指向new的对象  
    // All threads perform this; coordination is handled internally.
    rem_set()->younger_refs_iterate(perm_gen(), perm_blk);//perm的level是-1
  }
  _process_strong_tasks->all_tasks_completed();
}

```

看上面的`SH_PS_SystemDictionary_oops_do` task就知道了，这个就是对SystemDictionary进行扫描。

但是这里要说的是虽然有对SystemDictionary进行扫描，但是ygc的过程并不会对SystemDictionary进行处理，如果要对它进行处理需要开启类卸载的vm参数，CMS算法下，CMS GC和Full GC在开启CMSClassUnloadingEnabled的情况下是可能对类做卸载动作的，此时会对SystemDictionary进行清理，所以当我们在跑上面demo的时候，通过`jmap -dump:live,format=b,file=heap.bin <pid>`命令执行完之后，ygc的时间瞬间降下来了，不过又会慢慢回去，这是因为jmap的这个命令会做一次gc，这个gc过程会对SystemDictionary进行清理。

##修改VM代码验证
很遗憾hotspot目前没有对ygc的每个task做一个时间的统计，因此无法直接知道是不是`SH_PS_SystemDictionary_oops_do `这个task导致了ygc的时间变长，为了证明这个结论，我特地修改了一下代码，在上面的代码上加了一行：

```
if (!_process_strong_tasks->is_task_claimed(SH_PS_SystemDictionary_oops_do)) {
	GCTraceTime t("SystemDictionary_OOPS_DO",PrintGCDetails,true,NULL);
    if (so & SO_AllClasses) {
      SystemDictionary::oops_do(roots);
    } else if (so & SO_SystemClasses) {
      SystemDictionary::always_strong_oops_do(roots);
    }
  }
```

然后重新编译，跑我们的demo，测试结果如下：

```
2016-03-14T23:57:24.293+0800: [GC2016-03-14T23:57:24.294+0800: [ParNew2016-03-14T23:57:24.296+0800: [SystemDictionary_OOPS_DO, 0.0578430 secs]
: 81920K->3184K(92160K), 0.0889740 secs] 81920K->3184K(514048K), 0.0900970 secs] [Times: user=0.27 sys=0.00, real=0.09 secs]
2016-03-14T23:57:28.467+0800: [GC2016-03-14T23:57:28.468+0800: [ParNew2016-03-14T23:57:28.468+0800: [SystemDictionary_OOPS_DO, 0.0779210 secs]
: 85104K->5175K(92160K), 0.1071520 secs] 85104K->5175K(514048K), 0.1080490 secs] [Times: user=0.65 sys=0.00, real=0.11 secs]
2016-03-14T23:57:32.984+0800: [GC2016-03-14T23:57:32.984+0800: [ParNew2016-03-14T23:57:32.984+0800: [SystemDictionary_OOPS_DO, 0.1075680 secs]
: 87095K->8188K(92160K), 0.1434270 secs] 87095K->8188K(514048K), 0.1439870 secs] [Times: user=0.90 sys=0.01, real=0.14 secs]
2016-03-14T23:57:37.900+0800: [GC2016-03-14T23:57:37.900+0800: [ParNew2016-03-14T23:57:37.901+0800: [SystemDictionary_OOPS_DO, 0.1745390 secs]
: 90108K->7093K(92160K), 0.2876260 secs] 90108K->9992K(514048K), 0.2884150 secs] [Times: user=1.44 sys=0.02, real=0.29 secs]
```
我们会发现YGC的时间变长的时候，SystemDictionary_OOPS_DO的时间也会相应变长多少，因此验证了我们的说法。
