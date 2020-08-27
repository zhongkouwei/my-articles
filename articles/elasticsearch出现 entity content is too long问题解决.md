# elasticsearch出现 entity content is too long问题解决

## 一、现象

​	项目中使用了[elasticsearch]的Java Low Level REST Client来操作es，低级客户端的特性就是需要手动控制对es的请求，所以会遇到一些请求过程中的问题。比如今天，线上环境在导出数据时es报错，提示 entity content is too long [145648701] for the configured buffer limit。大概意思是返回值过大，超过buffer限制。在导出数据超过5000条时会出现该问题，导出数据较少时不会出现。

### 二、排查

先看一下低级客户端怎么存储返回值

```java
private final AtomicReference<Response> response = new AtomicReference<>();
```

```java
HttpAsyncResponseConsumerFactory DEFAULT = new HeapBufferedResponseConsumerFactory(DEFAULT_BUFFER_LIMIT);

static final int DEFAULT_BUFFER_LIMIT = 100 * 1024 * 1024;
```

​	可以看到，默认情况下，HeapBufferedResponseConsumerFactory限制的大小是100M。我们找一下能否配置。

```java
/**
 * Sends a request to the Elasticsearch cluster that the client points to. Blocks until the request is completed and returns
 * its response or fails by throwing an exception. Selects a host out of the provided ones in a round-robin fashion. Failing hosts
 * are marked dead and retried after a certain amount of time (minimum 1 minute, maximum 30 minutes), depending on how many times
 * they previously failed (the more failures, the later they will be retried). In case of failures all of the alive nodes (or dead
 * nodes that deserve a retry) are retried until one responds or none of them does, in which case an {@link IOException} will be thrown.
 *
 * This method works by performing an asynchronous call and waiting
 * for the result. If the asynchronous call throws an exception we wrap
 * it and rethrow it so that the stack trace attached to the exception
 * contains the call site. While we attempt to preserve the original
 * exception this isn't always possible and likely haven't covered all of
 * the cases. You can get the original exception from
 * {@link Exception#getCause()}.
 *
 * @param method the http method
 * @param endpoint the path of the request (without host and port)
 * @param params the query_string parameters
 * @param entity the body of the request, null if not applicable
 * @param httpAsyncResponseConsumerFactory the {@link HttpAsyncResponseConsumerFactory} used to create one
 * {@link HttpAsyncResponseConsumer} callback per retry. Controls how the response body gets streamed from a non-blocking HTTP
 * connection on the client side.
 * @param headers the optional request headers
 * @return the response returned by Elasticsearch
 * @throws IOException in case of a problem or the connection was aborted
 * @throws ClientProtocolException in case of an http protocol error
 * @throws ResponseException in case Elasticsearch responded with a status code that indicated an error
 * @deprecated prefer {@link #performRequest(Request)}
 */
@Deprecated
public Response performRequest(String method, String endpoint, Map<String, String> params,
                               HttpEntity entity, HttpAsyncResponseConsumerFactory httpAsyncResponseConsumerFactory,
                               Header... headers) throws IOException {
    Request request = new Request(method, endpoint);
    addParameters(request, params);
    request.setEntity(entity);
    setOptions(request, httpAsyncResponseConsumerFactory, headers);
    return performRequest(request);
}
```

​	可以看到，这个方法提供了一个可以配置HttpAsyncResponseConsumerFactory工厂类的参数，应该能符合我的要求。

```
    class HeapBufferedResponseConsumerFactory implements HttpAsyncResponseConsumerFactory {

        //default buffer limit is 100MB
        static final int DEFAULT_BUFFER_LIMIT = 100 * 1024 * 1024;

        private final int bufferLimit;

        public HeapBufferedResponseConsumerFactory(int bufferLimitBytes) {
            this.bufferLimit = bufferLimitBytes;
        }

        @Override
        public HttpAsyncResponseConsumer<HttpResponse> createHttpAsyncResponseConsumer() {
            return new HeapBufferedAsyncResponseConsumer(bufferLimit);
        }
    }
```

HeapBufferedResponseConsumerFactory(int bufferLimitBytes)这个方法可以修改默认的限制。所以我们只要将调用performRequest的代码改成调用这个方法即可，如下：

```java
HttpAsyncResponseConsumerFactory.HeapBufferedResponseConsumerFactory consumerFactory =
                new HttpAsyncResponseConsumerFactory.HeapBufferedResponseConsumerFactory(1024 * 1024 * 1024);

response = getRestClient().performRequest(method, url, params, httpEntity, consumerFactory);
```

### 三、超时问题

在进行了上述修改后，果然问题不再出现，但是重新请求，又出现一个报错：4,000 milliseconds timeout on connection http-outgoing-2 [ACTIVE]。 这个根据提示应该是超时了，因为返回数据过大，耗时比较久，加上我在初始化restClient时设置的timeout时间较短，所以需要修改一下配置。

```java
            RestClientBuilder.RequestConfigCallback requestConfigCallback = requestConfigBuilder -> {
                requestConfigBuilder.setConnectTimeout(300000);
                requestConfigBuilder.setSocketTimeout(400000);
                requestConfigBuilder.setConnectionRequestTimeout(0);
                return requestConfigBuilder;
            };
```

修改后问题不再出现。