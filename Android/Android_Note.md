1.ContentProvider 启动比 Application早

2.SharedPreferences#apply方法和主线程公用一个锁，容易导致ANR

3.查看某段代码附近的栈，可以通过new Exception(), 然后把Exception打印出来，来查看当前对应的方法栈
  或者通过此方法查看堆栈Thread.currentThread().getStackTrace()

4.Android 5.0之后通过JobService来应用保活，但是国内部分手机做了限制，譬如小米，JobService的最小间隔是60s

5.APP开启多个进程，可以增大进程的内存上限，但是每次都会走一下Application的onCreate方法

6.网上大片这样的网页，说5.0之后，跨应用启动Activity会启动新的任务栈，然而这条信息经过我验证之后是假的。。。。跨应用还是同一个任务栈，并且在近期任务中也还是显示的一个，并不是瞎传的2个 

7.当程序的compileSdkVersion SDK版本比较低的时候，此时想用高版本的API时，某种情况下可以使用反射调用新版本API，但某些时候是回调，不能直接使用反射，此时可以新建项目实现高版本接口，然后生成Jar文件，放到项目中即可。

