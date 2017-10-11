---
published: true
categories:
  - android
  - blog-post
description: Retrofit and RxJava.
image: assets/images/posts/error_handling.jpg
layout: post
date: '2017-10-10 11:51 +0000'
title: Custom Error Handling with RxJava & Retrofit 2
---
Recently I started redeveloping my Android app **Monitor for EnergyHive** in order to try and take advantage of some of the new techniques that "modern Android devs" are using. I must say, the many hours of reading up and research was totally worth it - MVP keeps everything much simpler, Retrofit stops me from having to build API calls myself and RxJava handles threading - letting me focus on the code instead of **avoiding the dreaded memory leaks**.

Before I added RxJava, in an effort to modularise things, I made use of **Retrofit Callbacks**. These had some extra logic that would take the erroneous server response and parse it into my own custom `Exception`. I would check for response error messages and error codes even when the server returned something other than code `2XX` and then create a **custom `ApiError` object**. I was able to create my own Callback that extended the Retrofit callback, and performed some logic in the Retrofit methods that passed the relevant `Exception` to two new abstract methods that would either return a custom response object (*parsed using GSON*) or one of my new `Exceptions`. I want to try and keep the blocks of code in this article to a minimum, so I'll just paste in the key code.

### ServiceCallback

<script src="https://gist.github.com/daniel-stoneuk/f1306d9ae0b6551027d4bca20081b12a.js"></script>

The **`ServiceCallback`** above takes the POJO that Retrofit will convert the JSON to. If the server sends a response, we can **handle the data in onResponse()** - for example we'll call the `success()` method if the HTTP Status code is 200. If this isn't the case, we'll call the `failure()` method passing a new instance of our custom `ApiException`. `ServiceResult` and `ServiceException` are both custom classes.

**`ServiceResult`** simply contains the parsed Object and the Okhttp Response object (in case response details are ever needed)

<script src="https://gist.github.com/daniel-stoneuk/1d8786154466e7fbc5963ccf66067d8a.js"></script>

We've made our own **subclass of `Exception`** that holds the message and cause. This form of Exception is called when there's an internet or other device related issue. ***If the server returns an error in its response, a subclass is returned (see below).***

<script src="https://gist.github.com/daniel-stoneuk/dde3739847563a7813aab896c6bd8b55.js"></script>

Let's say we make an call with incorrect parameters. The server will respond with an error message, and if it's a really good API, a **custom error code**.

<script src="https://gist.github.com/daniel-stoneuk/4f75a80aae73e2133c99c369e1fe10aa.js"></script>

Okay. So that's a lot of code so far - hopefully you're still with me. Seeing the code for yourself makes it slightly easier to follow I hope.

So now all that is done, we can actually use the `ServiceCallback`. We'll use it in place of `RetrofitCallback` in `.enqueue()`. We'll have the two methods - **success and failure** - we can override these and add our logic to. In the `failure` callback, we can check which type of `ServiceError` it is using `instanceof` and adjust our logic accordingly. We can even get the error message out of `ServiceApiException`.

<script src="https://gist.github.com/daniel-stoneuk/162a88389b220bcde570cf02c29de854.js"></script>

## RxJava

As I said earlier, ReactiveX is a popular way of making asynchronous calls, therefore if I'm trying to make my Java API complete, I'll want to include support for both basic **Callbacks and RxJava**.

I've structured my **singleton Service class** like this:

<script src="https://gist.github.com/daniel-stoneuk/c7fc853e2c6060942d54ef6c7f6b2500.js"></script>

The code above simplifies the process of making an API call, and it also mean that you can change the parameters before making the call. Usually Rx Calls don't send the response data to the onError callback. **We need to come up with our own way** of intercepting the response. The solution? Create our own `ErrorHandlingFunction` thats used in **`.onErrorResumeNext()`**

### MyRxErrorHandlerFunction
This takes a **copy of the `Observable`** and checks for all the different kinds of errors that are possible (for example, an HTTP Error). When each one is found it returns a new `Observable` with the one of our `ServiceException`'s. It looks similar to our custom Callback seen above.

<script src="https://gist.github.com/daniel-stoneuk/71d4f5350706850f09739e640ff1af46.js"></script>

There are other ways of intercepting the error, however the method shown above **should not affect any other operations** being performed on the `Observable`. Now we just need to add it to the Service class (since it was ommitted above). Also, in order to reduce further boilerplate code I have set the thread for `.subscribeOn()` here.

<script src="https://gist.github.com/daniel-stoneuk/fd61141f77645569d31d3bc1eaa069a5.js"></script>

Now we can use the `Observable` returned from the `authenticateRx` method like we would for any other `Observable`.

<script src="https://gist.github.com/daniel-stoneuk/b37660b4c830acc1b9181aa993f564b2.js"></script>

That's it. A lot of work for little gain, however, over time it should start to pay off. If the server has a certain error format you can adapt to it, if the response request changes you can change it in one place and be done with it. Etc. 

I no longer need to worry about parsing the errors and can just check the error code, which stays constant. Worth it in my opinion. Let me know if you end up taking inspiration through [Twitter - @daniel_stoneuk](https://twitter.com/daniel_stoneuk).

Dan
