title: Retrofit源码解析
date: 2018-05-31 16:47:33
category: 源码分析

tags: [代理模式,动态代理,http请求,retrofit]
---
#### 简介
[Retrofit](https://github.com/square/retrofit)是一个开源的http请求库，默认使用okhttp进行请求。它解耦，灵活（可以配置不同的http client来实现网络请求或配置不同的response解析工具）支持同步、异步和RxJava。而且使用简单，只需几个注解就可以构造一个http请求。

这里是基于2.4.0版本的源码进行解析。关于其使用方法可以参考[这里](https://www.jianshu.com/p/331f0bf161c2)。

#### 原理

Retrofit使用动态代理，在自定义接口方法调用时，先根据方法上的注解以及参数上的注解构造好一个ServiceMethod对象，该对象中存储了接口方法所包含的注解信息。然后构造OkHttpCall（默认是使用OKHttp），也就是真正发送请求的对象，其实现了Call接口，该接口是Retrofit定义的用来执行请求的接口（比如异步同步取消请求等）。最终，Retrofit会用适配器把Call接口（即OkHttpCall对象）转换为其他接口返回给使用者，比如默认的Call<?>接口或者支持rxjava的Obserable<?>接口。

当使用者触发Call<?>或者Obserable<?>请求时，适配器会触发Call接口（即OkHttpCall）调用，Call接口根据ServiceMethod信息构造OkHttp请求参数然后交由OkHttp发送请求。返回结果同理。

#### 一次请求

这里使用豆瓣接口，写一个请求的例子：

```java
 @OnClick(R.id.hello)
 public void onClickHello() {
        //构建Retrofit
        Retrofit retrofit = new Retrofit.Builder().baseUrl("https://api.douban.com/v2/")
                                                  .addConverterFactory(GsonConverterFactory.create()).
                                                          build();
        //创建动态代理
        GetBookDetail  getBookDetail = retrofit.create(GetBookDetail.class);
        //调用代理的方法，创建一个可以请求的对象
        Call<BookInfo> bookInfoCall  = getBookDetail.getBookInfo("1220562");
        //触发请求
        bookInfoCall.enqueue(new Callback<BookInfo>() {
            @Override
            public void onResponse(Call<BookInfo> call, Response<BookInfo> response) {
                BookInfo book = response.body();
                Log.e("book:", book.getSummary());
                hello.setText(book.getSummary());
            }

            @Override
            public void onFailure(Call<BookInfo> call, Throwable t) {
                Log.e("book:", "onFailure");
                t.printStackTrace();
            }
        });

 }
```




#### 动态代理
Retrofit之所以能够使用自定义的接口方法，而不去实现就能调用，使用的就是动态代理。动态代理可以动态的hook一个类的方法实现（方法必须是接口）。在代理行为已知，而代理接口不确定的时候使用，aop使用较多。

关于动态代理的介绍可以参考[这里]。

Retrofit.create方法创建动态代理：

```java
 public <T> T create(final Class<T> service) {
    //Retrofit要求service必须是接口
    Utils.validateServiceInterface(service);
    //validateEagerly表示提前验证service方法，默认是在方法invoke时处理。设置为true可以提前在这里处理。
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    //创建动态代理对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            //java8 支持接口方法含有默认实现，这里如果是默认方法，直接调用
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //解析注解
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            //真正的请求者
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //适配器转换接口
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```

#### 解析注解

解析注解主要是在ServiceMethod的build方法里面进行的，它分别解析方法上的注解以及参数上的注解，并且把相应信息保存在ServiceMethod中。

```java
public ServiceMethod build() {
      //创建适配器，后面使用
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      //创建返回结果转换器
      responseConverter = createResponseConverter();

      //解析方法上的注解
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      ...

      //解析参数上的注解
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        //解析每个参数注解，构建ParameterHandler
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }

	  ...

      return new ServiceMethod<>(this);
    }
```
parseMethodAnnotation根据不同的注解类型调用相应的方法进行解析。

这里只看下对GET注解的处理，最终调用到parseHttpMethodAndPath，该方法主要是保存相应的注解信息，还有解析url中使用“{}”的参数信息。

```java
 private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      if (this.httpMethod != null) {
        throw methodError("Only one HTTP method is allowed. Found: %s and %s.",
            this.httpMethod, httpMethod);
      }
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      if (value.isEmpty()) {
        return;
      }

      //不允许url中query参数
      // Get the relative URL path and existing query string, if present.
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // Ensure the query string does not have any named parameters.
        String queryParams = value.substring(question + 1);
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
          throw methodError("URL query string \"%s\" must not have replace block. "
              + "For dynamic query parameters use @Query.", queryParams);
        }
      }

      this.relativeUrl = value;
      //使用正则 解析url中的"{}"信息
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```
解析参数注解时，遍历方法参数注解，构造相应的ParameterHandler（每个参数注解都有一个实现），保存起来，用于在OkHttp构造请求时使用。ParameterHandler的作用是转换参数值并把参数值设置到RequestBuilder（用来构造OkHttp请求）中。

```java
private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      for (Annotation annotation : annotations) {
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);

        if (annotationAction == null) {
          continue;
        }

        if (result != null) {
          throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }

        result = annotationAction;
      }

      if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
      }

      return result;
    }


private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {
      
        ...
        else if (annotation instanceof Path) {
        if (gotQuery) {
          throw parameterError(p, "A @Path parameter must not come after a @Query.");
        }
        if (gotUrl) {
          throw parameterError(p, "@Path parameters may not be used with @Url.");
        }
        if (relativeUrl == null) {
          throw parameterError(p, "@Path can only be used with relative url on @%s", httpMethod);
        }
        gotPath = true;

        Path path = (Path) annotation;
        String name = path.value();
        validatePathName(p, name);
		//解耦转换参数值的逻辑  根据参数的类型和注解做不同的实现
        Converter<?, String> converter = retrofit.stringConverter(type, annotations);
        //返回相应的ParameterHandler
        return new ParameterHandler.Path<>(name, converter, path.encoded());

      } else if (annotation instanceof Query) {
        Query query = (Query) annotation;
        String name = query.value();
        boolean encoded = query.encoded();

        Class<?> rawParameterType = Utils.getRawType(type);
        gotQuery = true;
        if (Iterable.class.isAssignableFrom(rawParameterType)) {
          if (!(type instanceof ParameterizedType)) {
            throw parameterError(p, rawParameterType.getSimpleName()
                + " must include generic type (e.g., "
                + rawParameterType.getSimpleName()
                + "<String>)");
          }
          ParameterizedType parameterizedType = (ParameterizedType) type;
          Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
          Converter<?, String> converter =
              retrofit.stringConverter(iterableType, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded).iterable();
        } else if (rawParameterType.isArray()) {
          Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
          Converter<?, String> converter =
              retrofit.stringConverter(arrayComponentType, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded).array();
        } else {
          Converter<?, String> converter =
              retrofit.stringConverter(type, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded);
        }

      } 
	  ...
      }
```
#### 真正的请求者
创建完ServiceMethod后，相应的注解信息已经解析完成并保存到ServiceMethod中。下面创建真正发送请求的OkHttpCall。OkHttpCall实现了Call接口，当适配器转换后的外部接口触发请求的时候，最终会调用这里进行请求。看下异步请求的方法：

```java
@Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          //创建OkHttp的请求
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }
	//调用OkHttp进行异步请求
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          //解析结果  会调用serviceMethod.toResponse(catchingBody)，也就是利用serviceMethod中保存的GsonConverterFactory进行解析
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }

        try {
          //回调结果
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }
```
createRawCall调用到serviceMethod.toCall，该方法根据解析的注解信息，结合请求参数，构建OkHttp的请求对象。

```java
private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = serviceMethod.toCall(args);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }

okhttp3.Call toCall(@Nullable Object... args) throws IOException {
    //okhttp3构造Builder
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);

    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }

    for (int p = 0; p < argumentCount; p++) {
      //调用每个ParameterHandler，把参数值设置到requestBuilder中
      handlers[p].apply(requestBuilder, args[p]);
    }
    //创建okhttp3 的请求对象
    return callFactory.newCall(requestBuilder.build());
  }
```
#### 适配器
适配器会把标准Call接口转换为对外暴露的接口，比如Call<?>接口或者支持rxjava的Obserable<?>接口，来实现不同的触发方式或者结果回调逻辑。关于Obserable<?>适配器的实现会在后面补上。这里先分析下默认的适配器。

看下serviceMethod.adapt(okHttpCall)的实现：

```java
  T adapt(Call<R> call) {
    return callAdapter.adapt(call);
  }
```
跟踪callAdapter的创建，最终会到Retrofit.nextCallAdapter。而nextCallAdapter中的CallAdapter.Factory集合是在Retrofit.build中添加的。
```java

public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = callAdapterFactories.indexOf(skipPast) + 1;
    //从遍历工厂集合，从每个工厂中找到一个可以使用的
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }

    StringBuilder builder = new StringBuilder("Could not locate call adapter for ")
        .append(returnType)
        .append(".\n");
    if (skipPast != null) {
      builder.append("  Skipped:");
      for (int i = 0; i < start; i++) {
        builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
      }
      builder.append('\n');
    }
    builder.append("  Tried:");
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
    }
    throw new IllegalArgumentException(builder.toString());
  }


 public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
     //添加平台的默认适配器
      callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(1 + this.converterFactories.size());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
  }
```
看下Android平台的实现方法：
```java

static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    //android平台的适配器工厂
    @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
      if (callbackExecutor == null) throw new AssertionError();
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }

//android平台的适配器工厂
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
          //创建一个ExecutorCallbackCall
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```
ExecutorCallbackCall在异步请求时，在主线程中回调执行结果。它实现了call接口，并构造时传入了一个Call对象，这个对象是真实发送请求的对象，也就是前面的OkHttpCall对象。
```java
//实现了Call接口
static final class ExecutorCallbackCall<T> implements Call<T> {
    //在主线程执行任务
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
       //传入的call对象，这里即是OkHttpCall对象
      this.delegate = delegate;
    }

    //调用异步请求接口，即请求从这里开始
    @Override public void enqueue(final Callback<T> callback) {
      checkNotNull(callback, "callback == null");
      //调用代理call发送请求，即OkHttpCall对象
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
            //在android主线程执行回调
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }

    ...
  }
```

关键点

1. 动态代理
2. 注解的使用
3. 分层解耦


另：本人能力有限，如有错误，欢迎指正。