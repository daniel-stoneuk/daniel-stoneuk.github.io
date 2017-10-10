---
published: false
---
categories:
  - android
  - blog-post
description: It's even newer now.
image: assets/images/banner.jpg

#Custom Error Handling with RxJava & Retrofit 2.

Recently I started redeveloping my Android app** Monitor for EnergyHive** in order to try and take advantage of some of the new techniques that "modern Android devs" are using. I must say, the many hours of reading up and research was totally worth it - MVP keeps everything much simpler, Retrofit stops me from having to build API calls myself and RxJava handles threading - letting me focus on the code instead of **avoiding the dreaded memory leaks**.

Before I added RxJava, in an effort to modularise things, I made use of **Retrofit Callbacks**. These had some extra logic that would take the erroneous server response and parse it into my own custom `Exception`. I would check for response error messages and error codes even when the server returned something other than code `2XX` and then create a **custom `ApiError` object**. I was able to create my own Callback that extended the Retrofit callback, and performed some logic in the Retrofit methods that passed the relevant `Exception` to two new abstract methods that would either return a custom response object (*parsed using GSON*) or one of my new `Exceptions`. I want to try and keep the blocks of code in this article to a minimum, so I'll just paste in the key code.

###ServiceCallback

```java
public abstract class ServiceCallback<T> implements Callback<T> {
    @Override
    public void onResponse(Call<T> call, Response<T> response) {
        if (response.isSuccessful()) {
            success(new ServiceResult<T>(response.body(), response));
        } else {
            failure(new ServiceApiException(response));
        }
    }
    @Override
    public void onFailure(Call<T> call, Throwable t) {
        failure(new ServiceException("Request Failure", t));
    }
    public abstract void success(ServiceResult<T> result);
    public abstract void failure(ServiceException exception);
}
```

The **`ServiceCallback`** above takes the POJO that Retrofit will convert the JSON to. If the server sends a response, we can **handle the data in onResponse()** - for example we'll call the `success()` method if the HTTP Status code is 200. If this isn't the case, we'll call the `failure()` method passing a new instance of our custom `ApiException`. `ServiceResult` and `ServiceException` are both custom classes.

**`ServiceResult`** simply contains the parsed Object and the Okhttp Response object (in case response details are ever needed)

```java
public class ServiceResult<T> {
    public final T data;
    public final Response response;

    public ServiceResult(T data, Response response) {
        this.data = data;
        this.response = response;
    }
}
```

We've made our own **subclass of `Exception`** that holds the message and cause. This form of Exception is called when there's an internet or other device related issue. ***If the server returns an error in its response, a subclass is returned (see below).***

```java
public class ServiceException extends Exception {
    public ServiceException(String message) {
        super(message);
    }
	...
}
```

Let's say we make an call with incorrect parameters. The server will respond with an error message, and if it's a really good API, a **custom error code**.

```java
public class ServiceApiException extends ServiceException { 
    public ServiceApiException(Response response) {
        this(response, readApiError(response), DEFAULT_ERROR_CODE);
    }
    ServiceApiException(Response response, ApiError apiError, int apiErrorCode) {
        super(createExceptionMessage(apiErrorCode));
        this.apiError = apiError;
        this.code = apiErrorCode;
        this.response = response;
    }
    public static ApiError readApiError(Response response) {
        ...
        final String body = response.errorBody().source().buffer().clone().readUtf8();
        ...
        return parseApiError(body);
    }

    static ApiError parseApiError(String body) {
        ...
         final ApiError apiError = gson.fromJson(body, ApiError.class);
         // Return my own ApiError.
        ...
    }
	...
}
```

Okay. So that's a lot of code so far - hopefully you're still with me. Seeing the code for yourself makes it slightly easier to follow I hope.

So now all that is done, we can actually use the `ServiceCallback`. We'll use it in place of `RetrofitCallback` in `.enqueue()`. We'll have the two methods - **success and failure** - we can override these and add our logic to. In the `failure` callback, we can check which type of `ServiceError` it is using `instanceof` and adjust our logic accordingly. We can even get the error message out of `ServiceApiException`.

```java
.enqueue(new ServiceCallback<DeviceResponse>() {
    @Override
    public void success(ServiceResult<DeviceResponse> result) {
        // Success
    }

    @Override
    public void failure(ServiceException exception) {
        if (exception instanceof ServiceApiException) {
            final ServiceApiException ServiceApiException = (ServiceApiException) exception;
            setViewState(STATE_SERVICE_ERROR);
        } else {
            setViewState(STATE_INTERNET_ERROR);
        }
    }
});
```

##RxJava

As I said earlier, ReactiveX is a popular way of making asynchronous calls, therefore if I'm trying to make my Java API complete, I'll want to include support for both basic **Callbacks and RxJava**.

I've structured my **singleton Service class** like this:

```java
public class Service {

    Api api;

    public Service(Retrofit retrofit) {
        this.api = retrofit.create(Api.class);
    }


    public Call<LoginResponse> authenticate(String username, String password) {
        return api.authenticate(new LoginRequest(username, password));
    }

    public Observable<LoginResponse> authenticateRx(String username, String password) {
        ...
    }

    public Api getApi() {
        return api;
    }.

    public interface Api {
        @POST("auth")
        Call<LoginResponse> authenticate(@Body LoginRequest loginRequest);
        @POST("auth")
        Observable<LoginResponse> authenticateRx(@Body LoginRequest loginRequest);
    }
}
```

The code above simplifies the process of making an API call, and it also mean that you can change the parameters before making the call. Usually Rx Calls don't send the response data to the onError callback. **We need to come up with our own way** of intercepting the response. The solution? Create our own `ErrorHandlingFunction` thats used in **`.onErrorResumeNext()`**

###MyRxErrorHandlerFunction
This takes a **copy of the `Observable`** and checks for all the different kinds of errors that are possible (for example, an HTTP Error). When each one is found it returns a new `Observable` with the one of our `ServiceException`'s. It looks similar to our custom Callback seen above.

```java
  private class MyRxErrorHandlerFunction<T> implements Function<Throwable, ObservableSource<? extends T>> {
        @Override
        public ObservableSource<? extends T> apply(Throwable throwable) throws Exception {
            if (throwable instanceof HttpException) {
                HttpException httpException = (HttpException) throwable;
                if (httpException.response().isSuccessful()) {
                    return Observable.error(new ServiceException("Parsing error occurred"));
                } else {
                    return Observable.error(new ServiceApiException(httpException.response()));
                }
            }
            return Observable.error(new ServiceException("Connection Error - " + throwable.getMessage()));
        }
    }
```

There are other ways of intercepting the error, however the method shown above **should not affect any other operations** being performed on the `Observable`. Now we just need to add it to the Service class (since it was ommitted above). Also, in order to reduce further boilerplate code I have set the thread for `.subscribeOn()` here.

```java
public Observable<LoginResponse> authenticateRx(String username, String password) {
      return api.authenticateRx(new LoginRequest(username, password))
              .subscribeOn(Schedulers.io())
              .onErrorResumeNext(new MyRxErrorHandlerFunction<LoginResponse>());
}
```

Now we can use the `Observable` returned from the `authenticateRx` method like we would for any other `Observable`.

```java
  .getApiService()
  // Create the request
  .authenticateRx("daniel", "password")
  .observeOn(AndroidSchedulers.mainThread()) // Rx
  .subscribe(new Observer<ResourceReadingsResponse>() {
     @Override
     public void onSubscribe(Disposable d) {
		// Do what you like with the disposable (eg add it to a list)
     }
	 // Called when the response was successful and no errors occurred
     @Override
     public void onNext(LoginResponse response) {
     	useData(response)
     }

     @Override
     public void onError(Throwable e) {
   		if (e instanceof GlowApiException) {
  			GlowApiException glowApiException = (GlowApiException) e;
  			// Might log "Invalid Resource Id"
		} else if (e instanceof GlowException) {
  			GlowException glowException = ((GlowException) e);
  			// Might log "Connection Error"
		}
     }

     @Override
     public void onComplete() {
     }
  });
```

That's it. A lot of work for little gain, however, over time it should start to pay off. If the server has a certain error format you can adapt to it, if the response request changes you can change it in one place and be done with it. Etc. 

I no longer need to worry about parsing the errors and can just check the error code, which stays constant. Worth it in my opinion. Let me know if you end up taking inspiration through [Twitter - @daniel_stoneuk](https://twitter.com/daniel_stoneuk).

Dan

