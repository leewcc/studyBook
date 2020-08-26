# Exception 和 Error 的区别
Error：一般是指在 JVM 运行期间相关的严重错误，常见得Error类型有 OutOfMemmoryError 和 StackOverflowError

Exception：一般是指程序可处理的异常，
Java 的 Exception 分为 check exception 和 uncheck exception
-	check exception，需要显式进行捕获或者抛出。
-	uncheck exception 不需要显式处理，出现该异常大都是因为程序有 bug

常见的的 RumtimeException： 
NullPointerException
ArrayIndexOutOfBoundsException
ClassCastException
NumberFormatException
ArithmeticException
