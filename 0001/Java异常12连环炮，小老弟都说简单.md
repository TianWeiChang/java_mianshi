Java异常12连环炮，这次全了

大家好，面试连环炮系列，继续走起，今天给大家分享的Java异常面试连环跑。 

## 1、什么是Java中的异常？

异常是在程序执行期间可能发生的错误事件，并且会中断它的正常流程。异常可能来自不同类型的情况，例如用户输入的错误数据，硬件故障，网络连接故障等。

每当执行java语句时发生任何错误，都会创建一个异常对象，然后JRE会尝试查找异常处理程序来处理异常。如果找到合适的异常处理程序，则将异常对象传递给处理程序代码以处理异常，称为捕获异常。如果未找到处理程序，则应用程序将异常抛出到运行时环境，JRE将终止该程序。

Java异常处理框架仅用于处理运行时错误，编译时错误不由异常处理框架处理。

## 2、Java中的异常处理关键字是什么？

java异常处理中使用了四个关键字。

`throw`：有时我们明确要创建异常对象然后抛出它来停止程序的正常处理。throw关键字用于向运行时抛出异常来处理它。

`throws`：当我们在方法中抛出任何已检查的异常而不处理它时，我们需要在方法签名中使用throws关键字让调用者程序知道该方法可能抛出的异常。调用方法可以处理这些异常或使用throws关键字将其传播给它的调用方法。我们可以在`throws`子句中提供多个异常，也可以与main（）方法一起使用。

`try-catch`：我们在代码中使用try-catch块进行异常处理。try是块的开始，catch是在try块的末尾处理异常。我们可以使用try有多个catch块，try-catch块也可以嵌套。catch块需要一个应该是Exception类型的参数。

`finally`：finally块是可选的，只能用于try-catch块。由于异常会暂停执行过程，因此我们可能会打开一些不会关闭的资源，因此我们可以使用finally块。finally块总是被执行，无论是否发生异常。

## 3、解释Java异常层次结构？

Java异常是分层的，继承用于对不同类型的异常进行分类。`Throwable`是`Java Exceptions Hierarchy`的父类，它有两个子对象 - Error和Exception。异常进一步分为检查异常和运行时异常。

错误是超出应用程序范围的特殊情况，并且无法预测并从中恢复，例如硬件故障，JVM崩溃或内存不足错误。

Checked Exceptions是我们可以在程序中预期并尝试从中恢复的特殊情况，例如FileNotFoundException。我们应该捕获此异常并向用户提供有用的消息并正确记录以进行调试。Exception是所有Checked Exceptions的父类。

运行时异常是由错误的编程引起的，例如尝试从Array中检索元素。我们应该在尝试检索元素之前先检查数组的长度，否则它可能会ArrayIndexOutOfBoundException在运行时抛出。RuntimeException是所有运行时异常的父类。

## 4、Java异常类的重要方法是什么？

异常及其所有子类不提供任何特定方法，并且所有方法都在基类`Throwable`中定义。

`String getMessage()`

 - 此方法返回消息`String of Throwable`，并且可以在通过构造函数创建异常时提供消息。

`String getLocalizedMessage()`

 - 提供此方法，以便子类可以覆盖它以向调用程序提供特定于语言环境的消息。此方法`getMessage()`的可抛出类实现只是使用方法来返回异常消息。

`synchronized Throwable getCause()`

 - 此方法返回异常的原因或null id，原因未知。

`String toString()`

- 此方法以`String`格式返回有关`Throwable`的信息，返回的String包含`Throwable`类和本地化消息的名称。

`void printStackTrace()`

 - 此方法将堆栈跟踪信息打印到标准错误流，此方法已重载，我们可以将`PrintStream`或`PrintWriter`作为参数传递，以将堆栈跟踪信息写入文件或流。

## 5、解释Java 7 ARM功能和multi-catch块？**

如果你在一个`try`块中捕获了很多异常，你会发现`catch`块代码看起来非常难看，并且主要由冗余代码组成，以记录错误，记住`Java 7`的一个特性是`multi-catch`块。我们可以在一个`catch`块中捕获多个异常。具有此功能的catch块如下所示：

```java
try{
   //TODO ....
}catch(IOException | SQLException | Exception ex){
     logger.error(ex);
     throw new MyException(ex.getMessage());
}
```

大多数情况下，我们使用finally块来关闭资源，有时我们忘记关闭它们并在资源耗尽时获得运行时异常。这些异常很难调试，我们可能需要查看我们使用该类资源的每个地方，以确保我们关闭它。所以java 7的改进之一是try-with-resources，我们可以在try语句中创建一个资源并在try-catch块中使用它。当执行来自try-catch块时，运行时环境会自动关闭这些资源。具有这种改进的try-catch块样本是：

```java
try (MyResource mr = new MyResource()) {
        System.out.println("MyResource created in try-with-resources");
 } catch (Exception e) {
        e.printStackTrace();
}
```

## 6、Java中Checked和Unchecked Exception有什么区别？

`Checked Exceptions`应该使用`try-catch`块在代码中处理，否则方法应该使用throws关键字让调用者知道可能从方法抛出的已检查异常。未经检查的异常不需要在程序中处理或在方法的`throws`子句中提及它们。

`Exception`是所有已检查异常`RuntimeException`的超类，而是所有未经检查的异常的超类。请注意，`RuntimeException`是`Exception`的子类。

已检查的异常是需要在代码中处理的错误方案，否则您将收到编译时错误。例如，如果您使用FileReader读取文件，它会抛出FileNotFoundException，我们必须在try-catch块中捕获它或将其再次抛给调用方法。

未经检查的异常主要是由编程不良引起的，例如在对象引用上调用方法时的NullPointerException，而不确保它不为null。例如，我可以编写一个方法来从字符串中删除所有元音。确保不传递空字符串是调用者的责任。我可能会改变方法来处理这些场景，但理想情况下，调用者应该处理这个问题。

## 7、Java中throw和throws关键字有什么区别？

`throws`关键字与方法签名一起用于声明方法可能抛出的异常，而`throw`关键字用于破坏程序流并将异常对象移交给运行时来处理它。

## 8、如何在Java中编写自定义异常？

我们可以扩展`Exception`类或其任何子类来创建我们的自定义异常类。自定义异常类可以拥有自己的变量和方法，我们可以使用它们将错误代码或其他与异常相关的信息传递给异常处理程序。

自定义异常的一个简单示例如下所示。

```java
package com.journaldev.exceptions;

import java.io.IOException;

    public class MyException extends IOException {

    private static final long serialVersionUID = 4664456874499611218L;

    private String errorCode="Unknown_Exception";

    public MyException(String message, String errorCode){
        super(message);
        this.errorCode=errorCode;
    }

    public String getErrorCode(){
        return this.errorCode;
    }
}
```

## 9、Java中的OutOfMemoryError是什么？

`Java`中的`OutOfMemoryError`是`java.lang.VirtualMachineError`的子类，当`JVM`用完堆内存时，它会抛出它。我们可以通过提供更多内存来通过java选项运行java应用程序来修复此错误。

```shell
$>java MyProgram -Xms1024m -Xmx1024m -XX:PermSize=64M -XX:MaxPermSize=256m
```

## 10、主线程中的异常”有哪些不同的情况？

一些常见的主线程异常情况是：

**主线程java.lang.UnsupportedClassVersionError中的异常：**

当您的`Java`类是从另一个`JDK`版本编译并且您尝试从另一个`Java`版本运行它时，会出现此异常。

**主线程java.lang.NoClassDefFoundError中的异常：**

此异常有两种变体。第一个是您提供类全名和`.class`扩展名的地方。第二种情况是找不到`Class`。

**主线程java.lang.NoSuchMethodError中的异常：**

`main`：当您尝试运行没有main方法的类时会出现此异常。

**线程“main”中的异常java.lang.ArithmeticException：**

每当从`main`方法抛出任何异常时，它都会打印异常是控制台。第一部分解释了从main方法抛出异常，第二部分打印异常类名，然后在冒号后打印异常消息。

## 11、Java中的final，finally和finalize有什么区别？**

`final`和`finally`是`java`中的关键字，而`finalize`是一种方法。

`final`关键字可以与类变量一起使用，以便它们不能被重新分配，类可以避免按类扩展，并且使用方法来避免子类覆盖。

`finally`关键字与`try-catch`块一起使用，以提供始终执行的语句即使出现一些异常，通常最终也会用来关闭资源。

`finalize`方法由垃圾收集器在销毁对象之前执行，这是确保关闭所有全局资源的好方法。

在三者之中，最后只涉及到java异常处理。

## 12、当main方法抛出异常时会发生什么？

当main（）方法抛出异常时，Java Runtime终止程序并在系统控制台中打印异常消息和堆栈跟踪。

##13、我们可以有一个空的catch块吗？

我们可以有一个空的catch块，但它是最差编程的例子。我们永远不应该有空的catch块，因为如果异常被该块捕获，我们将没有关于异常的信息，并且它将成为调试它的噩梦。应该至少有一个日志记录语句来记录控制台或日志文件中的异常详细信息。

## 14、提供一些Java异常处理最佳实践？

与Java异常处理相关的一些最佳实践是：

1. 使用特定异常以便于调试。
2. 在程序中尽早抛出异常（Fail-Fast）。
3. 在程序后期捕获异常，让调用者处理异常。
4. 使用Java 7 ARM功能确保资源已关闭或使用finally块正确关闭它们。
5. 始终记录异常消息以进行调试。
6. 使用multi-catch块清洁关闭。
7. 使用自定义异常从应用程序API中抛出单一类型的异常。
8. 遵循命名约定，始终以Exception结束。
9. 记录在javadoc中使用@throws的方法抛出的异常。
10. 异常是昂贵的，所以只有在有意义的时候抛出它。否则，您可以捕获它们并提供空或空响应。



来自：www.journaldev.com





