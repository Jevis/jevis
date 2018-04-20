---
layout: post
title: Thread和Runable的解析
comments: true
description: 说起多线程，大家不会陌生啦，因为我们的项目中总是会用到。今天呢我看来很多大牛的博客，也渐渐的有了一个自己的理解，想把他们记录下来，和大家一起探讨一下。 
categories: android
author: jevis
---

## 综述：
&nbsp;&nbsp;说起多线程，大家不会陌生啦，因为我们的项目中总是会用到。今天呢我看了很多大牛的博客，也渐渐的有了一个自己的理解，想把他们记录下来，和大家一起探讨一下。

## Thread：
### 内容简介：
&nbsp;&nbsp;提到Thread就不得不先说说他的各种状态和他们之间的转化关系
![](/images/bolg_used/thread_runable/thread-state.png)
![](/images/bolg_used/thread_runable/thread-exchange-state.png)

 - NEW
   &nbsp;&nbsp;创建了一个Thread对象，但是还没有调用start来启动线程时。
 -  RUNNABLE
  &nbsp;&nbsp;调用了Thread的start方法，让该线程进入了可运行线程池中，等待线程被调度，获取cpu的使用权。
 - BLOCK
 &nbsp;&nbsp;线程被阻塞，通常我们遇到的是当线程A正在执行synchronize的方法时，线程B尝试进入这个方法区的时候就会发生阻塞。
 - WAITING
 &nbsp;&nbsp;当一个线程调用Object.wait，Thread.join的时候，当前的线程就会进入waiting状态，进入waiting状态就会释放CPU的执行权，并且释放资源.
 - TIMED_WAITING
  &nbsp;&nbsp;一个线程进入waiting状态，过一段时间之后再次进入runnable状态，一个线程可以通过Thread.sleep，Object.wait with timeout,Thread.join with timeput,LockSupport.parkNanos,LockSupport.parkUntil来进入TIME_WAITIN状态.
 - TERMINATED
 &nbsp;&nbsp;线程结束了就进入这个状态，也就是run执行完毕或者是因为异常而退出run方法

### WAITING和BLOCK的区别：
 &nbsp;&nbsp;大家看完了他们的运行状态可能有一个疑问，那就是WAITING和BLOCK的区别，那我就从代码和现实中两个方面来说一下我的见解：

#### 代码：

 ```
   synchronized(Obj){
        Obj.wait();
   }
 ```
  &nbsp;&nbsp;如果t1和t2两个线程先后执行了这个代码，t1先进去，执行了wait，这个时候t1就是waiting状态，而t2执行到synchronize的时候就卡住啦，t2的状态就叫做block。
  
### 现实：
 
  &nbsp;&nbsp;当你去一个已经满座的餐厅内吃饭的时候，里面的位置已经被占满啦，你只有等到里面的人出来你才能进去，这种情况下叫做block，你可以理解成不是因为你的问题，而是因为已经有人在里面执行（synchronize）啦，所以才造成你进不去的情况就做**block**。第二天你又去这个餐厅吃饭，结果发现里面很多位置，当你先进去的时候去发现你没有带钱人家不让进，这个时候你只能给你的朋友打电话让他来给你送钱过来，你在等待你盆友过来的情况叫做**waiting**状态，你的朋友给你送来钱（notify）,你才有权利去吃饭.
  
#### 使用:
&nbsp;&nbsp;通过集成Thread类来实现里面的run方法，调用的时候使用start方法，不要使用run方法来启动，因为run方法只是会像普通的方法那样执行，他不会开一个新的线程而是在调用方法时所在线程。
```
    class DemoThread extends Thread{
       @Override
       public void run() {
           super.run();
           //do something
       }
   }
   //调用时
    DemoThread thread = new DemoThread();
    thread.start();
    //thread.run();
```
 
## Runnable:
### 内容简介：
&nbsp;&nbsp;关于runnable的说明还是很少的，他是一个接口，要实现他的话呢就要实现里面的一个无参的run方法，是不是这个run方法在Thread中也有啊，那是当然啦，因为Thread就是实现的Runnable。
### 使用：
&nbsp;&nbsp;使用的方法也是很简单的，就是我们在前面说到的实现Runnabl接口，并在run方法中写上自己的逻辑。
```
     class PrimeRun implements Runnable {
         long minPrime;
         PrimeRun(long minPrime) {
             this.minPrime = minPrime;
         }

         public void run() {
             // compute primes larger than minPrime
              . . .
         }
     }
     //调用
        PrimeRun p = new PrimeRun(143);
        new Thread(p).start();
 
```
 
## Runnable和Thread的对比:
### 使用Runable接口的优势：
 1. 最明显的就是实现方式不同，一个是继承Thread类一个是实现Runnable接口，在java中只允许继承一个类然而却可以实现好多的接口，从而避免了继承带来的局限性。
 2. 还有一个好处就是能实现不同进程资源共享

&nbsp;&nbsp;那我们就来看第二个好处，也是我理解的时候坑了好一段时间，还是以卖票为例子
```
    class DemoThread extends Thread{
        private int ticket = 5;//总共5张票
        @Override
       public void run() {
           super.run();
           for (int i=0;i<10;i++)
           {
               if(ticket > 0){
                   Log.e(TAG, "thread: name="+Thread.currentThread().getName()+ " ticket = " + ticket--);
               }
               //线程sleep 让其他线程可以抢到cpu 从而执行
               try {
                   Thread.sleep(1);
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
       }
   }
```
&nbsp;&nbsp;注意看调用的方法
```
        //启用了三个线程 来执行卖票
        new DemoThread().start();
        new DemoThread().start();
        new DemoThread().start();
```
&nbsp;&nbsp;大家猜测一下输出也能猜到，输出了三边的5到1，因为我们使用了三个线程，每个线程里面都有5张票

     E/MainActivity: thread: name=Thread-4 ticket = 5
     E/MainActivity: thread: name=Thread-5 ticket = 5
     E/MainActivity: thread: name=Thread-6 ticket = 5
     E/MainActivity: thread: name=Thread-4 ticket = 4
     E/MainActivity: thread: name=Thread-5 ticket = 4
     E/MainActivity: thread: name=Thread-6 ticket = 4
     E/MainActivity: thread: name=Thread-4 ticket = 3
     E/MainActivity: thread: name=Thread-5 ticket = 3
     E/MainActivity: thread: name=Thread-6 ticket = 3
     E/MainActivity: thread: name=Thread-4 ticket = 2
     E/MainActivity: thread: name=Thread-5 ticket = 2
     E/MainActivity: thread: name=Thread-6 ticket = 2
     E/MainActivity: thread: name=Thread-4 ticket = 1
     E/MainActivity: thread: name=Thread-5 ticket = 1
     E/MainActivity: thread: name=Thread-6 ticket = 1

&nbsp;&nbsp;下面用Runnable来实现
```
    class DemoRunnable implements Runnable{
       private int ticket = 5;
       @Override
       public void run() {
           for (int i=0;i<10;i++)
           {
               if(ticket > 0){
                   Log.e(TAG, "runnable: name="+Thread.currentThread().getName()+ " ticket = " + ticket--);
               }
               try {
                   Thread.sleep(1);
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
       }
   }
```
&nbsp;&nbsp;在看这里的调用方式
```
//启用了三个线程，但却公用了一个Runable
        DemoRunnable runnable = new DemoRunnable();

        new Thread(runnable).start();
        new Thread(runnable).start();
        new Thread(runnable).start();
```
&nbsp;&nbsp;输出结果，在三个线程中共享了一个Runnable中的票数，可以理解为**在多线程中可以通过Runnable来实现数据共享**

    E/MainActivity: runnable: name=Thread-4 ticket = 5
    E/MainActivity: runnable: name=Thread-5 ticket = 4
    E/MainActivity: runnable: name=Thread-6 ticket = 3
    E/MainActivity: runnable: name=Thread-4 ticket = 2
    E/MainActivity: runnable: name=Thread-5 ticket = 1
    
&nbsp;&nbsp;肯定有人要说难道使用Thread就达不到这样的效果吗，可以的 只要你把你的Thread的执行方法写成这样，就可以达到一样的效果。
```
    DemoThread thread = new DemoThread();
    new Thread(thread).start();
    new Thread(thread).start();
    new Thread(thread).start();
```
&nbsp;&nbsp;不要忘记我们前面是说过的额Thread也是实现的Runnable的接口的 ，这两种写法其实都是调用的是Thread的Runnable的构造方法
```
public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }
```
&nbsp;&nbsp;我们还可以使用Runnable来实现输出3次5到1，只需要这样调用，相当于是3个线程里面有3个不同的runnable，所以他们之间的数据便没有任何的联系。
```
        new Thread(new DemoRunnable()).start();
        new Thread(new DemoRunnable()).start();
        new Thread(new DemoRunnable()).start();
```
&nbsp;&nbsp;所以总结一下我们可以使用Runnable接口来实现多线程内的数据共享，这样这句话就没有毛病啦 >.<。那么我们平时在使用多线程的时候是不是要用好多的new Thread的方法来写呢，当然不是啦！那样每次创建在销毁会很占用我们的资源的，所以我们当然是用 ThreadPoolExecutor 线程池了，那么关于线程池还有由于多线程操作数据造成的数据安全问题我们下面的博客慢慢讲！







