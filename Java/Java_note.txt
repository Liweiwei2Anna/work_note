1.对于ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue)
maximumPoolSize并非线程池最大运行线程数，这是一个误区
当线程数未到corePoolSize时，先填满corePoolSize对应的大小，当超过corePoolSize时，先填充workQueue,当workQueue时，再填满maximumPoolSize,超过这个数时，会被拒绝，抛出异常
这也是AsyncTask最大线程数有限制的原因

2.Exception不要乱用，Exception & Error的父类都是Throwable , 异常处理不了的情况，就应该抛出去，而不是自己catch，要让能处理的地方处理，现实中很多乱用的行为

3.Java的泛型是假泛型，在编译成class文件之后，会拿对应的实际类来替换


