---
layout: post
title: "JVM源码分析之栈溢出完全解读"
date: 2016-04-19 01:24:25 +0800
comments: true
categories: JVM StackOverflow
---
##概述
之所以想写这篇文章，其实是因为最近有不少系统出现了栈溢出导致进程crash的问题，并且很隐蔽，根本原因还得借助coredump才能分析出来，于是想从JVM实现的角度来全面分析下栈溢出的这类问题，或许你碰到过如下的场景:

* 日志里出现了StackOverflowError的异常
* 进程突然消失了，但是留下了crash日志
* 进程消失了，crash日志也没有留下

这些都可能是栈溢出导致的。
<!--more-->

##如何定位是否是栈溢出
上面提到的后面两种情况有可能不是我们今天要聊的栈溢出的问题导致的crash，也许是别的一些可能，那如何确定上面三种情况是栈溢出导致的呢？

* 出现了StackOverflowError，这种毫无疑问，必然是栈溢出，具体什么方法导致的栈溢出从栈上是能知道的，不过要提醒一点，我们打印出来看到的栈可能是不全的，因为JVM里对栈的输出条数是可以控制的，默认是1024，这个参数是`-XX:MaxJavaStackTraceDepth=1024`，可以将这个参数设置为-1，那将会全部输出对应的堆栈
* 如果进程消失了，但是留下了crash日志，那请检查下crash日志里的Current thread的stack范围，以及RSP寄存器的值，如果RSP寄存器的值是超出这个stack范围的，那说明是栈溢出了。
* 如果crash日志也没有留下，那只能通过coredump来分析了，在进程运行前，先执行`ulimit -c unlimited`，然后再跑进程，在进程挂掉之后，会产生一个`core.<pid>`的文件，然后再通过`jstack $JAVA_HOME/bin/java core.<pid>`来看输出的栈，如果正常输出了，那就可以看是否存在很长的调用栈的线程，当然还有可能没有正常输出的，因为jstack的这条从core文件抓栈的命令其实是基于serviceability agent来实现的，而SA在某些版本里是存在bug的，当然现在的SA也不能说完全没有bug，还是存在不少bug的，祝你好运。

##如何解决栈溢出的问题
这个需要具体问题具体分析，因为导致栈溢出的原因很多，提三个主要的：
* java代码写得不当，比如出现递归死循环，这也是最常见的，只能靠写代码的人稍微小心了
* native代码有栈上分配的逻辑，并且要求的内存还不小
* 线程栈空间设置比较小

有时候我们的代码需要调用到native里去，最常见的一种情况譬如`java.net.SocketInputStream.read0`方法，这是一个native方法，在进入到这个方法里之后，它首先就要求到栈上去分配一个64KB的缓存(64位linux)，试想一下如果执行到read0这个方法的时候，剩余的栈空间已经不足以分配64KB的内存了会怎样？也许就是一开头我们提到的crash，这只是一个例子，还有其他的一些native实现，包括我们自己也可能写这种native代码，如果真有这种情况，我们就需要好好斟酌下我们的线程栈到底要设置多大了。

如果我们的代码确实存在正常的很深的递归调用的话，通常是我们的栈可能设置太小，我们可以通过`-Xss`或者`-XX:ThreadStackSize`来设置java线程栈的大小，如果两个参数都设置了，那具体有效的是写在后面的那个生效。顺便提下，线程栈内存是和java heap独立的内存，并不是在java heap内分配的，是直接malloc分配的内存。

##线程栈大小

在jvm里，线程其实不仅仅只有一种，比如我们java里创建的叫做java线程，还有gc线程，编译线程等，默认情况下他们的栈大小如下：

{% codeblock lang:c %}
size_t os::Linux::default_stack_size(os::ThreadType thr_type) {
  // default stack size (compiler thread needs larger stack)
#ifdef AMD64
  size_t s = (thr_type == os::compiler_thread ? 4 * M : 1 * M);
#else
  size_t s = (thr_type == os::compiler_thread ? 2 * M : 512 * K);
#endif // AMD64
  return s;
}
{% endcodeblock %}
可见默认情况下编译线程需要的栈空间是其他种类线程的4倍。

各种类型的线程他们所需要的栈的大小其实是可以通过不同的参数来控制的：

{% codeblock lang:c %}
switch (thr_type) {
      case os::java_thread:
        // Java threads use ThreadStackSize which default value can be
        // changed with the flag -Xss
        assert (JavaThread::stack_size_at_create() > 0, "this should be set");
        stack_size = JavaThread::stack_size_at_create();
        break;
      case os::compiler_thread:
        if (CompilerThreadStackSize > 0) {
          stack_size = (size_t)(CompilerThreadStackSize * K);
          break;
        } // else fall through:
          // use VMThreadStackSize if CompilerThreadStackSize is not defined
      case os::vm_thread:
      case os::pgc_thread:
      case os::cgc_thread:
      case os::watcher_thread:
        if (VMThreadStackSize > 0) stack_size = (size_t)(VMThreadStackSize * K);
        break;
      }
{% endcodeblock %}

* `java_thread`的stack_size，其实就是-Xss或者-XX:ThreadStackSize的值
* `compiler_thread `的stack_size，是-XX:CompilerThreadStackSize指定的值
* vm内部的线程比如gc线程等可以通过-XX:VMThreadStackSize来设置


##JVM里栈溢出的实现
JVM里的栈溢出到底是怎么实现的，得从栈的大致结构说起：

{% codeblock lang:c %}
// Java thread:
//
//   Low memory addresses
//    +------------------------+
//    |                        |\  JavaThread created by VM does not have glibc
//    |    glibc guard page    | - guard, attached Java thread usually has
//    |                        |/  1 page glibc guard.
// P1 +------------------------+ Thread::stack_base() - Thread::stack_size()
//    |                        |\
//    |  HotSpot Guard Pages   | - red and yellow pages
//    |                        |/
//    +------------------------+ JavaThread::stack_yellow_zone_base()
//    |                        |\
//    |      Normal Stack      | -
//    |                        |/
// P2 +------------------------+ Thread::stack_base()
//
// Non-Java thread:
//
//   Low memory addresses
//    +------------------------+
//    |                        |\
//    |  glibc guard page      | - usually 1 page
//    |                        |/
// P1 +------------------------+ Thread::stack_base() - Thread::stack_size()
//    |                        |\
//    |      Normal Stack      | -
//    |                        |/
// P2 +------------------------+ Thread::stack_base()
//
// ** P1 (aka bottom) and size ( P2 = P1 - size) are the address and stack size returned from
//    pthread_attr_getstack()
{% endcodeblock %}

linux下java线程栈是从高地址往低地址方向走的，在栈尾（低地址）会预留两块受保护的内存区域，分别叫做yellow page和red page，其中yellow page在前，另外如果是java创建的线程，最后并没有图示的一个page的`glibc guard page`，非java线程是有的，但是没有yellow和red page，比如我们的gc线程，注意编译线程其实是java线程。

除了yellow page和red page，其实还有个shadow page，这三个page可以分别通过vm参数`-XX:StackYellowPages`,`-XX:StackRedPages`,`-XX:StackShadowPages`来控制。当我们要调用某个java方法的时候，它需要多大的栈其实是预先知道的，javac里就计算好了，但是如果调用的是native方法，那这就不好办了，在native方法里到底需要多大内存，这个无法得知，因此shadow page就是用来做一个大致的预测，看需要多大的栈空间，如果预测到新的RSP的值超过了yellowpage的位置，那就直接抛出栈溢出的异常，否则就去新的方法里处理，当我们的代码访问到yellow page或者red page里的地址的时候，因为这块内存是受保护的，所以会产生SIGSEGV的信号，此时会交给JVM里的信号处理函数来处理，针对yellow page以及red page会有不同的处理策略，其中yellow page的处理是会抛出StackOverflowError的异常，进程不会挂掉，也就是文章开头提到的第一个场景，但是如果是red page，那将直接导致进程退出，不过还是会产生Crash的日志，也就是文章开头提到的第二个场景，另外还有第三个场景，其实是没有栈空间了并且访问了超过了red page的地址，这个时候因为栈空间不够了，所以信号处理函数都进不去，因此就直接crash了，crash日志也不会产生。

{% codeblock lang:c %}
 if (sig == SIGSEGV) {
      address addr = (address) info->si_addr;

      // check if fault address is within thread stack
      if (addr < thread->stack_base() &&
          addr >= thread->stack_base() - thread->stack_size()) {
        // stack overflow
        if (thread->in_stack_yellow_zone(addr)) {
          thread->disable_stack_yellow_zone();
          if (thread->thread_state() == _thread_in_Java) {
            // Throw a stack overflow exception.  Guard pages will be reenabled
            // while unwinding the stack.
            stub = SharedRuntime::continuation_for_implicit_exception(thread, pc, SharedRuntime::STACK_OVERFLOW);
          } else {
            // Thread was in the vm or native code.  Return and try to finish.
            return 1;
          }
        } else if (thread->in_stack_red_zone(addr)) {
          // Fatal red zone violation.  Disable the guard pages and fall through
          // to handle_unexpected_exception way down below.
          thread->disable_stack_red_zone();
          tty->print_raw_cr("An irrecoverable stack overflow has occurred.");

          // This is a likely cause, but hard to verify. Let's just print
          // it as a hint.
          tty->print_raw_cr("Please check if any of your loaded .so files has "
                            "enabled executable stack (see man page execstack(8))");
        } else {
          // Accessing stack address below sp may cause SEGV if current
          // thread has MAP_GROWSDOWN stack. This should only happen when
          // current thread was created by user code with MAP_GROWSDOWN flag
          // and then attached to VM. See notes in os_linux.cpp.
          if (thread->osthread()->expanding_stack() == 0) {
             thread->osthread()->set_expanding_stack();
             if (os::Linux::manually_expand_stack(thread, addr)) {
               thread->osthread()->clear_expanding_stack();
               return 1;
             }
             thread->osthread()->clear_expanding_stack();
          } else {
             fatal("recursive segv. expanding stack.");
          }
        }
     }
  }
    
  ......
   
  if (stub != NULL) {
    // save all thread context in case we need to restore it
    if (thread != NULL) thread->set_saved_exception_pc(pc);

    uc->uc_mcontext.gregs[REG_PC] = (greg_t)stub;
    return true;
  }

  // signal-chaining
  if (os::Linux::chained_handler(sig, info, ucVoid)) {
     return true;
  }

  if (!abort_if_unrecognized) {
    // caller wants another chance, so give it to him
    return false;
  }

  if (pc == NULL && uc != NULL) {
    pc = os::Linux::ucontext_get_pc(uc);
  }

  // unmask current signal
  sigset_t newset;
  sigemptyset(&newset);
  sigaddset(&newset, sig);
  sigprocmask(SIG_UNBLOCK, &newset, NULL);

  VMError err(t, sig, pc, info, ucVoid);
  err.report_and_die();

  ShouldNotReachHere();   
{% endcodeblock %}
了解上面的场景之后，再回过头来想想JVM为什么要设置这几个page，其实是为了安全，能预测到栈溢出的话就抛出StackOverfolwError，而避免导致进程挂掉。



#欢迎各位关注个人微信公众号，主要围绕JVM写一系列的原理性，性能调优的文章

{% img /images/gzh.jpg 200 200 %}

