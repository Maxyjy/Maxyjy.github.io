---
title:  "Kotlin Coroutines"
layout: post
categories: Kotlin&Java
---

## 前言
Kotlin Coroutines 依赖于强大的 Kotlin Complier 编译器，将异步 Callback 回调式编程转化为同步代码写法，那其中是如何实现的呢 ？

// 回调嵌套
fun postItem(item: Item) {
    requestToken { token ->
        createPost(token, item) { post ->
            processPost(post)
        }
    }
}
 
// 协程
suspend fun postArticle(article: Article) {
    val token = requestToken()
    val post = createPost(token, article)
    processPost(post)
}
## Continuation Passing Style
简称CPS，翻译续体传递风格，好听的名字背后，实际上和 Callback 很相似

CPS的C，Continuation 只是一个接口，有两个方法 resume 和 resumeWithException，对应 Callback 的 onResponse、onFailure 方法。

interface Continuation<in T> {
   fun resume(result: Result<T>)
   fun resumeWithException(exception: Throwable)
}
查看 Kotlin 编译后的代码，可通过 Android Studio 工具栏 Tools -> Kotlin -> Show Kotlin Bytecode -> Decomplie 来查看大概的java 写法

写一个由 suspend 修饰的方法，编译后的方法签名都会自动生成一个额外的 Continuation 参数

// 原方法签名
suspend fun postArticle(article: Article) {...}
// 编译后的方法签名
fun postArticle(Article article, Continuation continuation) {...}
Continuation 的作用
让我们跟随一个简单的协程例子看起，现在有一个发布文章的 postArticle() 方法，但是需要通过网络获取 token后，再通过合并 article 和 token 来创建一个 post，最后发布 post，三个 suspend 方法串行执行。

suspend fun postArticle(article: Article) {
    // 获取token
    val token = requestToken()
    // 构造post
    val post = createPost(token, article)
    // 最后发布 post
    processPost(post)
}
//三个方法均为suspend修饰的方法
suspend fun requestToken(): String {...}
suspend fun createPost(token: String, article: Article): Post {...}
suspend fun processPost(post: Post) {...}
首先，Kotlin编译器会定义一个 int 类型属性 label 来记录执行到了哪个方法。

suspend fun postArticle(article: Article) {
    //label 0
    val token = requestToken()
    //label 1
    val post = createPost(token, article)
    //label 2
    processPost(post)
}
然后用一个大 switch 来判断 label，再执行对应方法。

fun postArticle(article: Article) {
    when (label) {
        case 0: { val token = requestToken(article) }
        case 1: { val post = createPost(token, article) }
        case 2: { processPost(post) }
    }
}
整体执行流程上，会执行多次postArticle()方法，每个方法执行完，label 会变成下一个方法的int值(可以认为是自增+1)，因为方法多次执行，所以 label 并不是方法内的局部变量，label 会存储在 stateMachine 状态机中，先不要被”状态机”这个高大上的名词吓到，状态机只是 Continuation 这个接口的实现。

    fun postArticle(article: Article, continuation: Continuation) {
        // 状态机
        val stateMachine = object :Continuation {
            var label: Int = 0
            ...
        }
        // 根据状态机的label来判断具体执行到了哪个方法
        when (label) {
            case 0: {
                stateMachine.label = 1
                requestToken(article, stateMachine)
            }
            case 1: {
                stateMachine.label = 2
                createPost(token, article, stateMachine)
            }
            case 2: {
                stateMachine.label = 3
                processPost(post, stateMachine)
            }
        }
    }
好了，现在有了 label 来判断具体是哪一个方法，那怎么实现方法重新执行？

还是利用状态机，每个方法最后一个参数是同个状态机的引用，状态机内有一个 resume() 方法，内部其实就是 postArticle() 方法，每个 suspend 方法执行完后，最后都会重新执行状态机的 resume() 方法。

    fun postArticle(article: Article, continuation: Continuation) {
        // 状态机
        val stateMachine = object :ContinuationImpl {
            var label: Int = 0
            ...
            fun resume() {
                // 每个方法执行完都会去 stateMachine.resume()
                postArticle(null, this)
            }
        }
        //根据状态机的label来判断具体执行到了哪个方法
        when (label) {
            case 0: {
                stateMachine.label = 1
                requestToken(article, stateMachine)
            }
            case 1: {
                stateMachine.label = 2
                createPost(token, article, stateMachine)
            }
            case 2: {
                stateMachine.label = 3
                processPost(post, stateMachine)
            }
        }
    }
” Wait a second…那你这个方法多次执行，不是每次都会创建一个状态机吗？”

其实并不会，之前我们说过，suspend 修饰的方法都会自动生成一个 Continuation 的参数，这个 Continuation 其实就是同个状态机对象。

执行 postArticle() 方法时，首先会判断传进来的 Continuation ，如果是状态机，并且不是null，就不会重新初始化了。

    fun postArticle(article: Article, continuation: Continuation) {
        // 判断传参进来的 continuation 是不是状态机实例
        val stateMachine = if (continuation is ContinuationImpl) {
            // 用传参的状态机
            continuation
        } else {
            // 首次执行，创建一个新的
            object : ContinuationImpl {
                var label: Int = 0
                var result: Object
                fun resume() {
                    postArticle(null, this)
                }
            }
        }
        when (label) {
            case 0: {
                stateMachine.label = 1
                requestToken(article, stateMachine)
            }
            case 1: {
                stateMachine.label = 2
                createPost(token, article, stateMachine)
            }
            case 2: {
                stateMachine.label = 3
                processPost(post, stateMachine)
            }
        }
    }
那方法的结果是如何获取的呢？其实也是存在状态机中的，状态机有 Object 类型的 result 来存储上一个方法的最终返回值，执行下个方法时会将 result 取出来转成对应类型。

    fun postArticle(article: Article, continuation: Continuation) {
        // 判断传参进来的 continuation 是不是状态机实例
        val stateMachine = if (continuation is ContinuationImpl) {
            // 用传参的状态机
            continuation
        } else {
            // 首次执行，创建一个新的
            object : ContinuationImpl {
                var label: Int = 0
                var result: Object
                fun resume() {
                    postArticle(null, this)
                }
            }
        }
        when (label) {
            case 0: {
                stateMachine.label = 1
                // requestToken 方法内部会对result进行保存
                // stateMachine.result = token
                requestToken(article, stateMachine)
            }
            case 1: {
                stateMachine.label = 2
                val token = stateMachine.result as String
                createPost(token, article, stateMachine)
            }
            case 2: {
                stateMachine.label = 3
                val post = stateMachine.result as Post
                processPost(post, stateMachine)
            }
        }
    }
## 总结
我理解的 CPS ，原理其实是协程内的 suspend 方法根据 label 区分，并将起始点方法 postArticle()，放在 Continuation 状态机对象的resume()方法中，然后将这个 Continuation 的引用传递给每个 suspend 方法，方法执行完后，label 发生变化，再通过 Continuation 的 resume() 方法去重新执行起始点 postArticle() 方法，逐步根据 label 顺序执行所有 suspend 方法。


该篇文章源自于笔者对 Kotlin 开发者大会中 Roman Elizarov 对 Kotlin Coroutines 原理讲解的理解

//https://www.youtube.com/watch?v=YrrUhttps://www.youtube.com/watch?v=YrrUCSi72E8CSi72E8
Youtube: KotlinConf 2017 – Deep Dive into Coroutines on JVM by Roman Elizarov