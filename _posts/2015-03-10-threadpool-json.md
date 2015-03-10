---
layout: post
title:  "threadpool"
date:   2015-03-10 00:18:23 
categories: threadpool
---

5种状态

```java

    /**
    * ctl是用来统计worker数量和线程池状态的，为了用一个int解决问题
    * workerCount 用低29位也，最大worker数量是(2^29)-1
    * runState 使用高3位
    **/
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
     
    private static final int RUNNING    = -1 << COUNT_BITS;  //111[29个0]
    private static final int SHUTDOWN   =  0 << COUNT_BITS;  //000[29个0]
    private static final int STOP       =  1 << COUNT_BITS;  //001[29个0]
    private static final int TIDYING    =  2 << COUNT_BITS;  //010[29个0]
    private static final int TERMINATED =  3 << COUNT_BITS;  //011[29个0]
```