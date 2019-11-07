# RestClient的TimeOut和Can't assign requested address问题排查及解决

## 问题背景

### TimeOut问题

在项目中用到了es，因为请求比较简单，所以使用了官方提供的Elasticsearch Java Low Level REST Client作为客户端，但在最近优化es操作的过程中，使用了并行操作+异步请求的方式，在请求数量较大时，performRequestAsync方法的失败回调中报错:TimeOut，第一反应是连接超时，但经过断点调试，连接并没有异常，在github上client项目的iisue中找到了原因，RestClient内部使用了连接池，在请求连接超时时会报错，所以TimeOut并不是连接超时，而是连接池超时的异常，通过在初始化client时设置参数为0的方式，可以关闭连接池超时机制。如下所示。

```java
RestClient restClient = RestClient.builder( )
  .setRequestConfigCallback(new RestClientBuilder.RequestConfigCallback() {
                @Override
                public RequestConfig.Builder customizeRequestConfig(RequestConfig.Builder requestConfigBuilder) {
                    requestConfigBuilder.setConnectTimeout(3000); // 连接超时
                    requestConfigBuilder.setSocketTimeout(4000); // 数据请求超时
                    requestConfigBuilder.setConnectionRequestTimeout(0); // 连接池超时设置
                    return requestConfigBuilder;
                }
            }).setMaxRetryTimeoutMillis(5*60*1000).build();
```

### Can't assign requested addres问题

TIME_WAIT状态

解决了连接超时的问题，再次运行时，发现同一时间的请求数量较大时，又抛出了Can't assign requested addres的异常，经过查阅，这个异常一般会在无可用端口时出现。在httpClient中出现这个异常一般是因为连接不是长连接，所以不断地断开重连，但关闭连接需要时间，所以在并发时出现无可用端口。我记得官方文档中提到了RestClient默认是长连接，但以防万一，在请求参数中加入了keepalive选项，结果问题不再出现，看来RestClient中默认并不是长连接。此处还需要继续考证。

```java
private static final RequestOptions COMMON_OPTIONS;

    static {
        RequestOptions.Builder builder = RequestOptions.DEFAULT.toBuilder();
        builder.addHeader("Connection", "keepalive");
        COMMON_OPTIONS = builder.build();
    }
```

