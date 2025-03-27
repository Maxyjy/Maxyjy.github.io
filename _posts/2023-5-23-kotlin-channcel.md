---
title:  "Kotlin Channel"
layout: post
categories: Kotlin&Java
---

Kotlin Flow 与操作符


## 一、Flow 流
Kotlin Flow 流实现实现异步的返回多个值

fun simple(): Flow<Int> = flow { // 流构建器
    for (i in 1..7) {
        delay(1000) // 假装我们在这里做了一些有用的事情
        println("emit $i")
        emit(i) // 发送下一个值
    }
}
simple().collect { value -> println(value) }
 
//注意 流是冷的，只有在开始收集的时候才会运行
//val flow = simple() //此时不会执行simple流中代码
//flow.collect { value -> println(value) } //此时会开始执行
有时我们需要在CPU线程进行计算，主线程收集流更新UI，但是flow {...} 构建器中的代码必须遵循上下文保存属性，并且不允许从其他上下文中发射（emit）。

fun main() = runBlocking {
    number().collect { value -> println(value) } //在Dispatchers.Main中收集流
}
 
fun number(): Flow<Int> = flow {
    withContext(Dispatchers.Default) { //在流构建器中改变了上下文为Dispatchers.Default
        for (i in 1..3) {
            Thread.sleep(1000L)
            emit(i) //抛出异常java.lang.IllegalStateException: Flow invariant is violated
        }
    }
}
不过，我们可以使用 flowOn 操作符正确的改变流在 emit 时的上下文。

注意这时的 collect 与 emit 发生在两个不同的线程中，flowOn 操作符创建了另一个协程。

fun main() = runBlocking {
    number().collect { value -> println("collect thread=" + Thread.currentThread().name + " value=$value") } //在Dispatchers.Main 中收集流
}
 
fun number(): Flow<Int> = flow { //在流构建器中改变了上下文为Dispatchers.Default 正确方式
    for (i in 1..3) {
        Thread.sleep(1000L)
        println("emit thread=" + Thread.currentThread().name)
        emit(i)
    }
}.flowOn(Dispatchers.Default)
## 二、操作符
### 1.过度流操作符
例如 map 、filter 过渡操作符应用于上游流，并返回下游流。操作符中的代码可以调用挂起函数。这些操作符也是冷操作符，就像流一样。这类操作符本身不是挂起函数。它运行的速度很快，返回新的转换流的定义。

suspend fun performRequest(request: Int): String {
    delay(1000) // 模仿长时间运行的异步工作
    return "response $request"
}
 
suspend fun analyzeResponse(response: String): String {
    delay(1000) // 模仿长时间运行的异步工作
    return "analyze result response = $response"
}
 
fun main() = runBlocking {
    (1..3).asFlow() // 一个请求流
        .map { request -> performRequest(request) } //从后端获取信息
        .map { request -> analyzeResponse(request) } //处理后段返回的结果
        .collect { result -> println(result) } //打印数据
}
#### 筛选操作符
fun main() = runBlocking {
    (1..5).asFlow()
        .filter {
            println("Filter $it") //筛选 1、2、3、4、5 中的偶数
            it % 2 == 0
        }
        .map {
            println("Map $it") //2、4
            "string $it"
        }.collect {
            println("Collect $it")//2、4
        }
}
#### 转换操作符
在流转换操作符中，最通用的一种 transform 。它可以用来模仿简单的转换，例如 map 与 filter，以及实施更复杂的转换。 使用 transform 操作符，我们可以 emit 任意值任意次。

fun main() = runBlocking {
    (1..3).asFlow() // 一个请求流
        .transform { request ->
            emit("Making request $request")
            emit(performRequest(request))
        }
        .collect { response -> println(response) }
}
#### 限长操作符
限长过渡操作符（例如 take）在流触及相应限制的时候会将它的执行取消。协程中的取消操作总是通过抛出异常来执行，这样所有的资源管理函数（如 try {...} finally {...} 块）会在取消的情况下正常运行：

fun numbers(): Flow<Int> = flow {
    try {
        emit(1)
        emit(2)
        emit(3)
    } finally {
        println("Finally in numbers")
    }
}
 
fun main() = runBlocking<Unit> {
    numbers()
        .take(2) // 只获取前两个，n(value) }
}
### 2、末端操作符
末端操作符是在流上用于启动流收集的挂起函数。 collect 是最基础的末端操作符，但是还有另外一些更方便使用的。

#### reduce 求和操作符
    //求和
    val sum = (1..5).asFlow()
        .map { it * it } // 数字 1 至 5 的平方 1、4、9、16、25
        .reduce { a, b -> a + b } // 求和（末端操作符）
    println(sum) //sum = 55
#### fold 带初始值求和
    //带初始值求和
    val fold = (1..5).asFlow()
        .map { it * it } // 数字 1 至 5 的平方
        .fold(100) { initial, value -> initial + value } // 求和并加上初始值100
    println(fold) //sum = 155
#### toList 转化为list
    //转为list
    val list = (1..5).asFlow()
        .map { it * it } // 数字 1 至 5 的平方
        .toList() // 转为list（末端操作符）
    list.forEach {
        println(it) //转化为list 1、4、9、16、25
    }
#### first 只收集第一个元素的
    //只收集第一个元素的流
    val first = (1..5).asFlow()
        .map { it } // 数字 5 的平方
        .first()
    println(first) //first = 1
#### single 只收集一个元素的流
    //限制只有一个元素的流 多于1个或null会抛出异常
    val single = flowOf(5)
        .map { it * it } // 数字 5 的平方
        .single()
    println(single) //single = 25
#### singleOrNull 限制只有一个元素或者为空的流
 
    //限制只有一个元素的流 多于1个会抛出异常 允许为null
    val singleOrNull = flowOf(5)
        .map { it * it } // 数字 5 的平方
        .singleOrNull()
    println(singleOrNull) //single = 25
    println(singleOrNull) //single = 25