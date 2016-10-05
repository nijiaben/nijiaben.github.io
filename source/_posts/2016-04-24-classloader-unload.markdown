---
layout: post
title: "JVM源码分析之JDK8下的僵尸(无法回收)类加载器"
date: 2016-04-24 11:21:24 +0800
comments: true
categories: JVM ClassLoader JDK8
---
##概述
这篇文章基于最近在排查的一个问题，花了我们团队不少时间来排查这个问题，现象是有一些类加载器是作为key放到WeakHashMap里的，但是经历过多次full gc之后，依然坚挺地存在内存里，但是从代码上来说这些类加载器是应该被回收的，因为没有任何强引用可以到达这些类加载器了，于是我们做了内存dump，分析了下内存，发现除了一个WeakHashMap外并没有别的GC ROOT途径达到这些类加载器了，那这样一来经过多次FULL GC肯定是可以被回收的，但是事实却不是这样，为了让这个问题听起来更好理解，还是照例先上个Demo，完全模拟了这种场景。

<!--more-->
##Demo
首先我们创建两个类AAA和AAB，分别打包到两个不同jar里，比如AAA.jar和AAB.jar，这两个类之间是有关系的，AAA里有个属性是AAB类型的，注意这两个jar不要放到classpath里让appClassLoader加载到：

```
public class AAA {
        private AAB aab;
        public AAA(){
                aab=new AAB();
        }
        public void clear(){
                aab=null;
        }
}

public class AAB {}
```

接着我们创建一个类加载TestLoader，里面存一个WeakHashMap，专门来存TestLoader的，并且复写loadClass方法，如果是加载AAB这个类，就创建一个新的TestLoader来从AAB.jar里加载这个类

```
import java.net.URL;
import java.net.URLClassLoader;
import java.util.WeakHashMap;


public class TestLoader extends URLClassLoader {
        public static WeakHashMap<TestLoader,Object> map=new WeakHashMap<TestLoader,Object>();
        private static int count=0;
        public TestLoader(URL[] urls){
                super(urls);
                map.put(this, new Object());
        }
        @SuppressWarnings("resource")
        public Class<?> loadClass(String name) throws ClassNotFoundException {
                if(name.equals("AAB") && count==0){
                        try {
                                count=1;
                    URL[] urls = new URL[1];
                    urls[0] = new URL("file:///home/nijiaben/tmp/AAB.jar");
                    return new TestLoader(urls).loadClass("AAB");
                }catch (Exception e){
                    e.printStackTrace();
                }
                }else{
                        return super.loadClass(name);
                }
                return null;
        }
}
```

再看我们的主类TTest，一些说明都写在类里了：

```
import java.lang.reflect.Method;
import java.net.URL;

/**
 * Created by nijiaben on 4/22/16.
 */
public class TTest {
    private Object aaa;
    public static void main(String args[]){
        try {
            TTest tt = new TTest();
            //将对象移到old，并置空aaa的aab属性
            test(tt);
            //清理掉aab对象
            System.gc();
            System.out.println("finished");
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    @SuppressWarnings("resource")
        public static void test(TTest tt){
        try {
        	//创建一个新的类加载器，从AAA.jar里加载AAA类
            URL[] urls = new URL[1];
            urls[0] = new URL("file:///home/nijiaben/tmp/AAA.jar");
            tt.aaa=new TestLoader(urls).loadClass("AAA").newInstance();
            //保证类加载器对象能进入到old里，因为ygc是不会对classLoader做清理的
            for(int i=0;i<10;i++){
                System.gc();
                Thread.sleep(1000);
            }
            //将aaa里的aab属性清空掉，以便在后面gc的时候能清理掉aab对象，这样AAB的类加载器其实就没有什么地方有强引用了，在full gc的时候能被回收
            Method[] methods=tt.aaa.getClass().getDeclaredMethods();
            for(Method m:methods){
                if(m.getName().equals("clear")){
                        m.invoke(tt.aaa);
                        break;
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

运行的时候请跑在JDK8下，打个断点在`System.out.println("finished")`的地方，然后做一次内存dump。

从上面的例子中我们得知，TTest是类加载器AppClassLoader加载的，其属性aaa的对象类型是通过TestLoader从AAA.jar里加载的，而aaa里的aab属性是从一个全新的类加载器TestLoader从AAB.jar里加载的，当我们做了多次System GC之后，这些对象会移到old，在做最后一次GC之后，aab对象会从内存里移除，其类加载器此时已经是没有任何地方的强引用了，只有一个WeakHashMap引用它，理论上做GC的时候也应该被回收，但是事实时这个AAB的这个类加载器并没有被回收，从分析结果来看，GC ROOT路径是WeakHashMap，如图所示：

{% img /images/2016/04/mat.png %}


##JDK8里的metaspace
这里不得不提的一个概念是JDK8里的metaspace，它是为了取代perm的，至于好处是什么，我个人觉得不是那么明显，有点费力不讨好的感觉，代码改了很多，但是实际收益并不明显，据说是oracle内部斗争的一个结果。

在JDK8里虽然没了perm，但是klass的信息还是要有地方存，jvm里为此分配了两块内存，一块是紧挨着heap来的，就和perm一样，专门用来存klass的信息，可以通过`-XX:CompressedClassSpaceSize`来设置大小，另外一块和它们不一定连着，主要是存非klass之外的其他信息，比如常量池什么的，可以通过`-XX:InitialBootClassLoaderMetaspaceSize`来设置，同时我们还可以通过`-XX:MaxMetaspaceSize`来设置触发metaspace回收的阈值。

每个类加载器都会从全局的metaspace空间里取一些metaChunk管理起来，当有类定义的时候，其实就是从这些内存里分配的，当不够的时候再去全局的metaspace里分配一块并管理起来。

这块具体的情况后面可以专门写一篇文章来介绍，包括内存结构，内存分配，GC等。

##JDK8里的ClassLoaderDataGraph
每个类加载器都会对应一个ClassLoaderData的数据结构，里面会存譬如具体的类加载器对象，加载的klass，管理内存的metaspace等，它是一个链式结构，会链到下一个ClassLoaderData上，gc的时候通过ClassLoaderDataGraph来遍历这些ClassLoaderData，ClassLoaderDataGraph的第一个ClassLoaderData是bootstrapClassLoader的

```
class ClassLoaderData : public CHeapObj<mtClass> {
  ...
  static ClassLoaderData * _the_null_class_loader_data;

  oop _class_loader;          // oop used to uniquely identify a class loader
                              // class loader or a canonical class path
  Dependencies _dependencies; // holds dependencies from this class loader
                              // data to others.

  Metaspace * _metaspace;  // Meta-space where meta-data defined by the
                           // classes in the class loader are allocated.
  Mutex* _metaspace_lock;  // Locks the metaspace for allocations and setup.
  bool _unloading;         // true if this class loader goes away
  bool _keep_alive;        // if this CLD is kept alive without a keep_alive_object().
  bool _is_anonymous;      // if this CLD is for an anonymous class
  volatile int _claimed;   // true if claimed, for example during GC traces.
                           // To avoid applying oop closure more than once.
                           // Has to be an int because we cas it.
  Klass* _klasses;         // The classes defined by the class loader.

  JNIHandleBlock* _handles; // Handles to constant pool arrays

  // These method IDs are created for the class loader and set to NULL when the
  // class loader is unloaded.  They are rarely freed, only for redefine classes
  // and if they lose a data race in InstanceKlass.
  JNIMethodBlock*                  _jmethod_ids;

  // Metadata to be deallocated when it's safe at class unloading, when
  // this class loader isn't unloaded itself.
  GrowableArray<Metadata*>*      _deallocate_list;

  // Support for walking class loader data objects
  ClassLoaderData* _next; /// Next loader_datas created

  // ReadOnly and ReadWrite metaspaces (static because only on the null
  // class loader for now).
  static Metaspace* _ro_metaspace;
  static Metaspace* _rw_metaspace;

  ...

}
```

这里提几个属性：

* `_class_loader ` : 就是对应的类加载器对象
* `_keep_alive` : 如果这个值是true，那这个类加载器会认为是活的，会将其做为GC ROOT的一部分，gc的时候不会被回收
* `_unloading` : 表示这个类加载是否需要卸载的
* `_is_anonymous` : 是否匿名，这种ClassLoaderData主要是在lambda表达式里用的，这个我后面会详细说
* `_next` : 指向下一个ClassLoaderData，在gc的时候方便遍历
* `_dependencies` : 这个属性也是本文的重点，后面会细说

再来看下构造函数：

```
ClassLoaderData::ClassLoaderData(Handle h_class_loader, bool is_anonymous, Dependencies dependencies) :
  _class_loader(h_class_loader()),
  _is_anonymous(is_anonymous),
  // An anonymous class loader data doesn't have anything to keep
  // it from being unloaded during parsing of the anonymous class.
  // The null-class-loader should always be kept alive.
  _keep_alive(is_anonymous || h_class_loader.is_null()),
  _metaspace(NULL), _unloading(false), _klasses(NULL),
  _claimed(0), _jmethod_ids(NULL), _handles(NULL), _deallocate_list(NULL),
  _next(NULL), _dependencies(dependencies),
  _metaspace_lock(new Mutex(Monitor::leaf+1, "Metaspace allocation lock", true)) {
    // empty
}
```
可见，`_keep_ailve`属性的值是根据`_is_anonymous`以及当前类加载器是不是bootstrapClassLoader来的。

`_keep_alive`到底用在哪？其实是在GC的的时候，来决定要不要用Closure或者用什么Closure来扫描对应的ClassLoaderData。

```
void ClassLoaderDataGraph::roots_cld_do(CLDClosure* strong, CLDClosure* weak) {
  //从最后一个创建的classloader到bootstrapClassloader  
  for (ClassLoaderData* cld = _head;  cld != NULL; cld = cld->_next) {
    //如果是ygc，那weak和strong是一样的，对所有的类加载器都做扫描，保证它们都是活的 
    //如果是cms initmark阶段，如果要unload_classes了(should_unload_classes()返回true)，则weak为null，那就只遍历bootstrapclassloader以及正在做匿名类加载的类加载  
    CLDClosure* closure = cld->keep_alive() ? strong : weak;
    if (closure != NULL) {
      closure->do_cld(cld);
    }
  }
```


##类加载器什么时候被回收
类加载器是否需要被回收，其实就是看这个类加载器对象是否是活的，所谓活的就是这个类加载器加载的任何一个类或者这些类的对象是强可达的，当然还包括这个类加载器本身就是GC ROOT一部分或者有GC ROOT可达的路径，那这个类加载器就肯定不会被回收。

从各种GC情况来看：

* 如果是YGC，类加载器是作为GC ROOT的，也就是都不会被回收
* 如果是Full GC，只要是死的就会被回收
* 如果是CMS GC，CMS GC过程也是会做标记的（这是默认情况，不过可以通过一些参数来改变），但是不会做真正的清理，真正的清理动作是发生在下次进入安全点的时候。

##僵尸类加载器如何产生
如果类加载器是与GC ROOT的对象存在真正依赖的这种关系，这种类加载器对象是活的无可厚非，我们通过zprofiler或者mat都可以分析出来，可以将链路绘出来，但是有两种情况例外：


###lambda匿名类加载
lambda匿名类加载走的是unsafe的defineAnonymousClass方法，这个方法在vm里对应的是下面的方法

```
UNSAFE_ENTRY(jclass, Unsafe_DefineAnonymousClass(JNIEnv *env, jobject unsafe, jclass host_class, jbyteArray data, jobjectArray cp_patches_jh))
{
  instanceKlassHandle anon_klass;
  jobject res_jh = NULL;

  UnsafeWrapper("Unsafe_DefineAnonymousClass");
  ResourceMark rm(THREAD);

  HeapWord* temp_alloc = NULL;

  anon_klass = Unsafe_DefineAnonymousClass_impl(env, host_class, data,
                                                cp_patches_jh,
                                                   &temp_alloc, THREAD);
  if (anon_klass() != NULL)
    res_jh = JNIHandles::make_local(env, anon_klass->java_mirror());

  // try/finally clause:
  if (temp_alloc != NULL) {
    FREE_C_HEAP_ARRAY(HeapWord, temp_alloc, mtInternal);
  }

  // The anonymous class loader data has been artificially been kept alive to
  // this point.   The mirror and any instances of this class have to keep
  // it alive afterwards.
  if (anon_klass() != NULL) {
    anon_klass->class_loader_data()->set_keep_alive(false);
  }

  // let caller initialize it as needed...

  return (jclass) res_jh;
}
UNSAFE_END
}

```
可见，在创建成功匿名类之后，会将对应的ClassLoaderData的`_keep_alive`属性设置为false，那是不是意味着`_keep_alive`属性在这之前都是true呢？下面的`parse_stream`方法是从上面的方法最终会调下来的方法

```
Klass* SystemDictionary::parse_stream(Symbol* class_name,
                                      Handle class_loader,
                                      Handle protection_domain,
                                      ClassFileStream* st,
                                      KlassHandle host_klass,
                                      GrowableArray<Handle>* cp_patches,
                                      TRAPS) {
  TempNewSymbol parsed_name = NULL;

  Ticks class_load_start_time = Ticks::now();

  ClassLoaderData* loader_data;
  if (host_klass.not_null()) {
    // Create a new CLD for anonymous class, that uses the same class loader
    // as the host_klass
    assert(EnableInvokeDynamic, "");
    guarantee(host_klass->class_loader() == class_loader(), "should be the same");
    guarantee(!DumpSharedSpaces, "must not create anonymous classes when dumping");
    loader_data = ClassLoaderData::anonymous_class_loader_data(class_loader(), CHECK_NULL);
    loader_data->record_dependency(host_klass(), CHECK_NULL);
  } else {
    loader_data = ClassLoaderData::class_loader_data(class_loader());
  }

  instanceKlassHandle k = ClassFileParser(st).parseClassFile(class_name,
                                                             loader_data,
                                                             protection_domain,
                                                             host_klass,
                                                             cp_patches,
                                                             parsed_name,
                                                             true,
                                                             THREAD);
...

}

ClassLoaderData* ClassLoaderData::anonymous_class_loader_data(oop loader, TRAPS) {
  // Add a new class loader data to the graph.
  return ClassLoaderDataGraph::add(loader, true, CHECK_NULL);
}

ClassLoaderData* ClassLoaderDataGraph::add(Handle loader, bool is_anonymous, TRAPS) {
  // We need to allocate all the oops for the ClassLoaderData before allocating the
  // actual ClassLoaderData object.
  ClassLoaderData::Dependencies dependencies(CHECK_NULL);

  No_Safepoint_Verifier no_safepoints; // we mustn't GC until we've installed the
                                       // ClassLoaderData in the graph since the CLD
                                       // contains unhandled oops

  ClassLoaderData* cld = new ClassLoaderData(loader, is_anonymous, dependencies);

...
}
```

从上面的代码得知，只要走了unsafe的那个方法，都会为当前类加载器创建一个ClassLoaderData对象，并设置其`_is_anonymous`为true，也同时意味着`_keep_alive`的属性是true，并加入到ClassLoaderDataGraph中。

试想如果创建的这个匿名类没有成功，也就是`anon_klass()==null`，那这个`_keep_alive`属性就永远无法设置为false了，这意味着这个ClassLoaderData对应的ClassLoader对象将永远都是GC ROOT的一部分，无法被回收，这种情况就是真正的僵尸类加载器了，不过目前我还没模拟出这种情况来，有兴趣的同学可以试一试，如果真的能模拟出来，这绝对是JDK里的一个BUG，可以提交给社区。


###类加载器依赖导致的
这里说的类加载器依赖，并不是说ClassLoader里的parent建立的那种依赖关系，如果是这种关系，那其实通过mat或者zprofiler这样的工具都是可以分析出来的，但是还存在一种情况，那些工具都是分析不出来的，这种关系就是通过ClassLoaderData里的`_dependencies`属性得出来的，比如说如果A类加载器的`_dependencies`属性里记录了B类加载器，那当GC遍历A类加载器的时候也会遍历B类加载器，并将其标活，哪怕B类加载器其实是可以被回收了的，可以看下下面的代码

```
void ClassLoaderData::oops_do(OopClosure* f, KlassClosure* klass_closure, bool must_claim) {
  if (must_claim && !claim()) {
    return;
  }

  f->do_oop(&_class_loader);
  _dependencies.oops_do(f);
  _handles->oops_do(f);
  if (klass_closure != NULL) {
    classes_do(klass_closure);
  }
}
```

那问题来了，这种依赖关系是怎么记录的呢？其实我们上面的demo就模拟了这种情况，可以仔细去看看，我也针对这个demo描述下，比如加载AAA的类加载器TestLoader加载AAA后，并创建AAA对象，此时会看到有个类型是AAB的属性，此时会对常量池里的类型做一个解析，我们看到TestLoader的loadClass方法的时候做了一个判断，如果是AAB类型的类加载，那就创建一个新的类加载器对象从AAB.jar里去加载，当加载返回的时候，在jvm里其实就会记录这么一层依赖关系，认为AAA的类加载器依赖AAB的类加载器，并记录下来，但是纵观所有的hotspot代码，并没有一个地方来清理这种依赖关系的，也就是说只要这种依赖关系建立起来，会一直持续到AAA的类加载器被回收的时候，AAB的类加载器才会被回收，所以说这算一种伪僵尸类加载器，虽然从依赖关系上其实并不依赖了(比如demo里将AAA的aab属性做clear清空动作)，但是GC会一直认为他们是存在这种依赖关系的，会持续存在一段时间，具体持续多久就看AAA类加载器的情况了。

针对这种情况个人认为需要一个类似引用计数的GC策略，当某两个类加载器确实没有任何依赖的时候，将其清理掉这种依赖关系，估计要实现这种改动的地方也挺多，没那么简单，所以当时的设计者或许因为这样并没有这么做了，我觉得这算是偷懒妥协的结果吧，当然这只是我的一种猜测。

#欢迎各位关注个人微信公众号，主要围绕JVM写一系列的原理性，性能调优的文章

{% img /images/gzh.jpg 200 200 %}
