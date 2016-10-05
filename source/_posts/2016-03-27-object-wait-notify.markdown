---
layout: post
title: "JVM源码分析之Object.wait/notify(All)完全解读"
date: 2016-03-27 11:31:25 +0800
comments: true
categories: JVM 
---
##概述
本文其实一直都想写，因为各种原因一直拖着没写，直到开公众号的第一天，有朋友再次问到这个问题，这次让我静心下来准备写下这篇文章，本文有些东西是我自己的理解，比如为什么JDK一开始要这么设计，初衷是什么，没怎么去找相关资料，所以只能谈谈自己的理解，所以大家看到文章之后可以谈谈自己的看法，对于实现部分我倒觉得说清楚问题不大，code is here，看明白了就知道怎么回事了。

<!--more-->
Object.wait/notify(All)大家都知道主要是协同线程处理的，大家用得也很多，大概逻辑和下面的用法差不多

```
        new Thread(){
            public void run(){
                synchronized (lock){
                    try{
                        lock.wait();
                    }catch (InterruptedException e){
                        e.printStackTrace();
                    }
                }
            }
        }.start();

        new Thread(){
            public void run(){
                synchronized (lock){
                    lock.notify();
                }
            }
        }.start();
```

看到上面代码，你会有什么疑惑吗？至少我会有几个问题会问自己：

* 为什么进入wait和notify的时候要加synchronized锁
* 既然加了synchronized锁，那当某个线程调用了wait的时候明明还在synchronized块里，其他线程怎么进入到锁里去执行notify的
* 为什么wait方法可能会抛出InterruptedException异常
* 如果有多个线程都进入wait状态，那某个线程调用notify唤醒线程时是否按照顺序唤起那些wait线程
* wait的线程是在某个线程执行完notify之后立马就被唤起吗
* notifyAll又是怎么实现全唤起的
* wait的线程是否会影响load

如果上面这些问题也都是你想了解的，那这篇文章或许能给你一个答案。

##为何要加synchronized锁
从实现上来说，这个锁至关重要，正因为这把锁，才能让整个wait/notify玩转起来，当然我觉得其实通过其他的方式也可以实现类似的机制，不过hotspot至少是完全依赖这把锁来实现wait/notify的。

如果要我们来实现这种机制我们会怎么去做，我们知道wait/notify是为了线程间协作而设计的，当我们执行wait的时候让线程挂起，当执行notify的时候唤醒其中一个挂起的线程，那需要有个地方来保存对象和线程之间的映射关系(可以想象一个map，key是对象，value是一个线程列表)，当调用这个对象的wait方法时，将当前线程放到这个线程列表里，当调用这个对象的notify方法时从这个线程列表里取出一个来让其继续执行，这样看来是可行的，也比较简单，那现在的问题这种映射关系放到哪里。而synchronized正好也是为线程间协作而设计的，上面碰到的问题它也要解决，或许正因为这样wait和notify的实现就直接依赖synchronzied(monitorenter/monitorexit是jvm规范里要求要去实现的)来实现了，这只是我的理解，可能初衷不是这个原因，这其实也是这篇文章迟迟未写的一个原因吧，因为我无法取证自己的理解是对的，欢迎各位在这块谈谈自己的见解。

##wait方法执行后未退出同步块，其他线程如何进入同步块
这个问题其实要回答很简单，因为在wait处理过程中会临时释放同步锁，不过需要注意的是当某个线程调用notify唤起了这个线程的时候，在wait方法退出之前会重新获取这把锁，只有获取了这把锁才会继续执行，想象一下，我们知道wait的方法是被monitorenter和monitorexit包围起来，当我们在执行wait方法过程中如果释放了锁，出来的时候又不拿锁，那在执行到monitorexit指令的时候会发生什么？当然这可以做兼容，不过这实现起来还是很奇怪的。

##为什么wait方法可能抛出InterruptedException异常
这个异常大家应该都知道，当我们调用了某个线程的interrupt方法时，对应的线程会抛出这个异常，wait方法也不希望破坏这种规则，因此就算当前线程因为wait一直在阻塞，当某个线程希望它起来继续执行的时候，它还是得从阻塞态恢复过来，因此wait方法被唤醒起来的时候会去检测这个状态，当有线程interrupt了它的时候，它就会抛出这个异常从阻塞状态恢复过来。

这里有两点要注意：

* 如果被interrupt的线程只是创建了，并没有start，那等他start之后进入wait态之后也是不能会恢复的
* 如果被interrupt的线程已经start了，在进入wait之前，如果有线程调用了其interrupt方法，那这个wait等于什么都没做，会直接跳出来，不会阻塞


##被notify(All)的线程有规律吗
这里要分情况：

* 如果是通过notify来唤起的线程，那先进入wait的线程会先被唤起来
* 如果是通过nootifyAll唤起的线程，默认情况是最后进入的会先被唤起来，即LIFO的策略

##notify执行之后立马唤醒线程吗
其实这个大家可以验证一下，在notify之后写一些逻辑，看这些逻辑是在其他线程被唤起之前还是之后执行，这个是个细节问题，可能大家并没有关注到这个，其实hotspot里真正的实现是退出同步块的时候才会去真正唤醒对应的线程，不过这个也是个默认策略，也可以改的，在notify之后立马唤醒相关线程。

##notifyAll是怎么实现全唤起的
或许大家立马想到这个简单，一个for循环就搞定了，不过在jvm里没实现这么简单，而是借助了monitorexit，上面我提到了当某个线程从wait状态恢复出来的时候，要先获取锁，然后再退出同步块，所以notifyAll的实现是调用notify的线程在退出其同步块的时候唤醒起最后一个进入wait状态的线程，然后这个线程退出同步块的时候继续唤醒其倒数第二个进入wait状态的线程，依次类推，同样这这是一个策略的问题，jvm里提供了挨个直接唤醒线程的参数，不过都很罕见就不提了。

##wait的线程是否会影响load
这个或许是大家比较关心的话题，因为关乎系统性能问题，wait/nofity是通过jvm里的park/unpark机制来实现的，在linux下这种机制又是通过pthread_cond_wait/pthread_cond_signal来玩的，因此当线程进入到wait状态的时候其实是会放弃cpu的，也就是说这类线程是不会占用cpu资源。


#欢迎各位关注个人微信公众号，主要围绕JVM写一系列的原理性，性能调优的文章

{% img /images/gzh.jpg 200 200 %}
