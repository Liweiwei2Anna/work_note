1.ContentProvider 启动比 Application早

2.SharedPreferences#apply方法和主线程公用一个锁，容易导致ANR

3.查看某段代码附近的栈，可以通过new Exception(), 然后把Exception打印出来，来查看当前对应的方法栈
  或者通过此方法查看堆栈Thread.currentThread().getStackTrace()

4.Android 5.0之后通过JobService来应用保活，但是国内部分手机做了限制，譬如小米，JobService的最小间隔是60s

5.APP开启多个进程，可以增大进程的内存上限，但是每次都会走一下Application的onCreate方法

