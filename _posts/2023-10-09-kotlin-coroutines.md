---
title:  "What can we do with Kotlin Coroutines?"
mathjax: false
layout: post
categories: Kotlin&Java
---

## What can we do with Kotlin Coroutines?

### 引言

我们知道，在Android中，在主线程中执行耗时较长的异步任务是不合适的，这会阻塞处理UI事件的主线程，在Android4.0后，会抛出NetworkOnMainThreadException()来限制开发者在主线程中做网络请求。

这时，我们需要为网络请求创建一个工作子线程，例如直接使用Thread或者使用Java Executors。

```javascript
    //Thread
    thread {
        //network request
        getComment()
    }
 
    //Java Executors
    Executors.newCachedThreadPool().execute {
        //network request
        getComment()
    }
```

然后在getComment()方法中，则需要这样请求接口。

```javascript
    private fun getComment() {
        apiClient.getComment("example post id")?.enqueue(object : Callback<CommentModel> {
            override fun onResponse(call: Call<CommentModel>, response: Response<CommentModel>) {
                //将请求到的数据显示到TextView上
                textView.text = response.body().toString()
            }
 
            override fun onFailure(call: Call<CommentModel>, t: Throwable) {
                //将错误信息显示到TextView上
                textView.text = "network error"
            }
        })
    }
```

### 开始使用协程

使用协程，不再需要创建线程，取而代之的是创建一个协程。

```javascript
    //Kotlin Coroutines
    //Dispatcher.IO 指定协程运行在用来执行磁盘或网络I/O的调度器中
    lifecycleScope.launch(Dispatchers.IO) {
        getComment()
    }
```

而getComment方法中，代码可以得到极大的简化，不再需要使用Callback接口来接收请求结果。

```javascript
    private suspend fun getComment() {
        val apiClient = CommentApiClient()
        try {
            val comment = apiClient.getComment("example post id")
            withContext(Dispatchers.Main) { //切换到主线程
                textView.text = comment?.body //请求成功
            }
        } catch (e: NetworkErrorException) {
            textView.text = "network error" //请求失败
        }
    }
```

### 使用协程处理串行任务

如果有一个业务场景，需要我们先请求用户的ID，再使用用户的ID来获得评论呢？

可能我们会这样写，请求UserInfo返回结果后，在onResponse回调中再请求Comment，然后在getComment的onResponse方法中更新UI，标准的嵌套地狱。不过，这仅仅是2个请求串行，如果是5个…10个…

```javascript
    //先请求UserInfo
    apiClient.getUserInfo()?.enqueue(object : Callback<UserInfo> {
        override fun onResponse(call: Call<UserInfo>, response: Response<UserInfo>) {
        //处理UserInfo
        ...
        //使用用户id作为参数，仔请求获取Comment
        apiClient.getComment(info.userId)?.enqueue(object : Callback<CommentModel> {            
             override fun onResponse(call: Call<CommentModel>, response: Response<CommentModel>) {
             //将请求到Comment的数据显示到TextView上
             textView.text = response.body().toString()
        ...
    }
```

但，使用协程，只需要这样。

```javascript
    //请求用户信息
    val userInfo = apiClient.getUserInfo()
    //使用用户信息中的id作为参数，请求用户评论
    val comment = apiClient.getComment(userInfo.userId)
    withContext(Dispatchers.Main) { //切换到主线程
        textView.text = comment?.body //请求成功
    }
```


### 使用协程处理并发任务

如果有一个业务场景，需要我们请求用户个人信息和用户头像，然后将请求到的两个结果在同一时间展示出来，要怎么做？我们可能想到直接请求两个接口，然后再把两个请求的结果进行融合。

不过，将两个结果融合很难做到，也许我们会使用CountDownLatch、CompletableFuture，或者更好用的Rxjava来处理。

但在协程里，只需要这样。


```javascript
    //请求用户昵称
    val userName = apiClient.getUserName()
    //请求用户头像
    val userAvatar = apiClient.getUserAvatar()
    //切换到主线程
    withContext(Dispatchers.Main) {
        //请求成功
        updateUI(userName.await(),userAvatar.await())
    }
```

*请求用户昵称* 、*请求用户头像* 都是异步操作，但使用协程可以将代码写出非异步的风格。



