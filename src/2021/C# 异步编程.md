# 异步异常

异步对异常的支持和对其他方面的支持类似，你可以像同步代码那样顺序地编写正常执行的代码和异常抛出代码。Task不能成功执行完毕时，他们会抛出异常。客户端（调用Task）的代码如果正在await一个已经开始执行的Task，那么他会捕获到这个异常。

注意，当一个异步执行的Task抛出一个异常，这个Task会是Faulted状态。这个Task对象将在Task.Exception属性中持有这个异常。如果Faulted状态的Task被await，他将会抛出一个异常。

> NOTE
>
> 如果异步方法的返回类型为void，则异常将不会被.Net Framework捕获和处理，如果你也没有在异步方法内部进行捕获，那么异常将形同于未处理异常。线程中的未处理异常，将会造成非常严重的后果。
>
> 如果异常方法的返回类型是Task族，但是调用处没有进行await（直接await），则异常会存在于Task.Exception中，

现在仍然有两个重要的机制需要理解：异常如何在Faulted Task中存储，以及一个异常如何被取出并重新在await这个Task的代码中抛出。

异步的代码抛出的异常存储在这个异步的Task中。Task.Exception的类型是System.AggregateException，这是因为在异步工作中，可能有多个异常被抛出，这些异常都会被添加到AggregateException.InnerExceptions这个集合中。

最常见的Faulted Task的情景是，Task仅包含一个异常。当代码await一个Faulted Task时，第一个异常将会被取出并在调用处重新抛出，注意，这里抛出的是实际发生的异常，而不是AggregateException。抽取第一个异常抛出使得异步的异常处理机制近似于同步代码。如果你的代码有更精细的处理机制，可以对Task.Exception进行检查。

# 异常示例说明

### 返回类型为void的异步方法

```C#
// 异步方法
public async void FuncAsync()
{
    throw new Exception("1");
}

// 调用代码
public int main()
{
    try
    {
        FuncAsync();
    }
    catch (Exception e)
    {
        // 捕获不到，异常将直接在异步方法中抛出
    }
}
```

