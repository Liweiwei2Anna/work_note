1.输出LOG的时候，往往需要输入TAG，其实是可以省去的，通过
StackTraceElement ele = new Throwable().fillInStackTrace().getStackTrace()[index];
String.format("[%s,%d,%s] %s", ele.getFileName(), ele.getLineNumber(), ele.getMethodName(), msg);
其中index通过方法栈的深度来确定值，来输入LOG 
