# URL
  - https://www.jianshu.com/p/f38a62efaa96
  
  接着上一篇，我们在使用HttpClient的时候更多的是需要自己根据业务特点创建自己定制化的HttpClient实例，而不是像之前那样使用

// 使用默认配置创建httpclient的实例

````
CloseableHttpClient client = HttpClients.createDefault();
废话不多说，直接上代码(Talk is cheap, show me the code!)：

import org.apache.http.HttpEntity;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

/**
 * 使用httpclient-4.5.2发送请求，配置请求定制化参数
 * @author chmod400
 * 2016.3.28
 */
public class RequestConfigDemo {

    public static void main(String[] args) {
        try {
            String url = "http://www.baidu.com";
            /**
             *  请求参数配置
             *  connectionRequestTimeout:
             *                          从连接池中获取连接的超时时间，超过该时间未拿到可用连接，
             *                          会抛出org.apache.http.conn.ConnectionPoolTimeoutException: Timeout waiting for connection from pool
             *  connectTimeout:
             *                  连接上服务器(握手成功)的时间，超出该时间抛出connect timeout
             *  socketTimeout:
             *                  服务器返回数据(response)的时间，超过该时间抛出read timeout
             */
            CloseableHttpClient client = HttpClients.custom().setDefaultRequestConfig(RequestConfig.custom()
                    .setConnectionRequestTimeout(2000).setConnectTimeout(2000).setSocketTimeout(2000).build()).build();
            
            HttpPost post = new HttpPost(url);
//          HttpGet get = new HttpGet(url);
            
            CloseableHttpResponse response = client.execute(post);
//          CloseableHttpResponse response = client.execute(get);
            
            // 服务器返回码
            int status_code = response.getStatusLine().getStatusCode();
            System.out.println("status_code = " + status_code);
            
            // 服务器返回内容
            String respStr = null;
            HttpEntity entity = response.getEntity();
            if(entity != null) {
                respStr = EntityUtils.toString(entity, "UTF-8");
            }
            System.out.println("respStr = " + respStr);
            // 释放资源
            EntityUtils.consume(entity);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
````

需要说明的是，需要自己定制HttpClient客户端的话，我们使用HttpClients.custom()，然后调用各种set方法即可，一般建议使用HttpClients.custom().setDefaultRequestConfig()，org.apache.http.client.config.RequestConfig类提供了很多可定制的参数，我们可以根据自己的配置来使用相关配置。有几个参数我们自己必须设置一下

connectionRequestTimeout:从连接池中获取连接的超时时间，超过该时间未拿到可用连接，会抛出org.apache.http.conn.ConnectionPoolTimeoutException: Timeout waiting for connection from pool
connectTimeout:连接上服务器(握手成功)的时间，超出该时间抛出connect timeout
socketTimeout:服务器返回数据(response)的时间，超过该时间抛出read timeout
通过打断点的方式我们知道，HttpClients在我们没有指定连接工厂的时候默认使用的是连接池工厂org.apache.http.impl.conn.PoolingHttpClientConnectionManager.PoolingHttpClientConnectionManager(Registry<ConnectionSocketFactory>)，所以我们需要配置一下从连接池获取连接池的超时时间。

以上3个超时相关的参数如果未配置，默认为-1，意味着无限大，就是一直阻塞等待！

官方提供了一个demo，里面有一些最常用的配置代码，仅供参考：


````
/*
 * ====================================================================
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 * ====================================================================
 *
 * This software consists of voluntary contributions made by many
 * individuals on behalf of the Apache Software Foundation.  For more
 * information on the Apache Software Foundation, please see
 * <http://www.apache.org/>.
 *
 */

package org.apache.http.examples.client;

import java.net.InetAddress;
import java.net.UnknownHostException;
import java.nio.charset.CodingErrorAction;
import java.util.Arrays;

import javax.net.ssl.SSLContext;

import org.apache.http.Consts;
import org.apache.http.Header;
import org.apache.http.HttpHost;
import org.apache.http.HttpRequest;
import org.apache.http.HttpResponse;
import org.apache.http.ParseException;
import org.apache.http.client.CookieStore;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.client.config.AuthSchemes;
import org.apache.http.client.config.CookieSpecs;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.protocol.HttpClientContext;
import org.apache.http.config.ConnectionConfig;
import org.apache.http.config.MessageConstraints;
import org.apache.http.config.Registry;
import org.apache.http.config.RegistryBuilder;
import org.apache.http.config.SocketConfig;
import org.apache.http.conn.DnsResolver;
import org.apache.http.conn.HttpConnectionFactory;
import org.apache.http.conn.ManagedHttpClientConnection;
import org.apache.http.conn.routing.HttpRoute;
import org.apache.http.conn.socket.ConnectionSocketFactory;
import org.apache.http.conn.socket.PlainConnectionSocketFactory;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.impl.DefaultHttpResponseFactory;
import org.apache.http.impl.client.BasicCookieStore;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.DefaultHttpResponseParser;
import org.apache.http.impl.conn.DefaultHttpResponseParserFactory;
import org.apache.http.impl.conn.ManagedHttpClientConnectionFactory;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.apache.http.impl.conn.SystemDefaultDnsResolver;
import org.apache.http.impl.io.DefaultHttpRequestWriterFactory;
import org.apache.http.io.HttpMessageParser;
import org.apache.http.io.HttpMessageParserFactory;
import org.apache.http.io.HttpMessageWriterFactory;
import org.apache.http.io.SessionInputBuffer;
import org.apache.http.message.BasicHeader;
import org.apache.http.message.BasicLineParser;
import org.apache.http.message.LineParser;
import org.apache.http.ssl.SSLContexts;
import org.apache.http.util.CharArrayBuffer;
import org.apache.http.util.EntityUtils;

/**
 * This example demonstrates how to customize and configure the most common aspects
 * of HTTP request execution and connection management.
 */
public class ClientConfiguration {

    public final static void main(String[] args) throws Exception {

        // Use custom message parser / writer to customize the way HTTP
        // messages are parsed from and written out to the data stream.
        HttpMessageParserFactory<HttpResponse> responseParserFactory = new DefaultHttpResponseParserFactory() {

            @Override
            public HttpMessageParser<HttpResponse> create(
                SessionInputBuffer buffer, MessageConstraints constraints) {
                LineParser lineParser = new BasicLineParser() {

                    @Override
                    public Header parseHeader(final CharArrayBuffer buffer) {
                        try {
                            return super.parseHeader(buffer);
                        } catch (ParseException ex) {
                            return new BasicHeader(buffer.toString(), null);
                        }
                    }

                };
                return new DefaultHttpResponseParser(
                    buffer, lineParser, DefaultHttpResponseFactory.INSTANCE, constraints) {

                    @Override
                    protected boolean reject(final CharArrayBuffer line, int count) {
                        // try to ignore all garbage preceding a status line infinitely
                        return false;
                    }

                };
            }

        };
        HttpMessageWriterFactory<HttpRequest> requestWriterFactory = new DefaultHttpRequestWriterFactory();

        // Use a custom connection factory to customize the process of
        // initialization of outgoing HTTP connections. Beside standard connection
        // configuration parameters HTTP connection factory can define message
        // parser / writer routines to be employed by individual connections.
        HttpConnectionFactory<HttpRoute, ManagedHttpClientConnection> connFactory = new ManagedHttpClientConnectionFactory(
                requestWriterFactory, responseParserFactory);

        // Client HTTP connection objects when fully initialized can be bound to
        // an arbitrary network socket. The process of network socket initialization,
        // its connection to a remote address and binding to a local one is controlled
        // by a connection socket factory.

        // SSL context for secure connections can be created either based on
        // system or application specific properties.
        SSLContext sslcontext = SSLContexts.createSystemDefault();

        // Create a registry of custom connection socket factories for supported
        // protocol schemes.
        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
            .register("http", PlainConnectionSocketFactory.INSTANCE)
            .register("https", new SSLConnectionSocketFactory(sslcontext))
            .build();

        // Use custom DNS resolver to override the system DNS resolution.
        DnsResolver dnsResolver = new SystemDefaultDnsResolver() {

            @Override
            public InetAddress[] resolve(final String host) throws UnknownHostException {
                if (host.equalsIgnoreCase("myhost")) {
                    return new InetAddress[] { InetAddress.getByAddress(new byte[] {127, 0, 0, 1}) };
                } else {
                    return super.resolve(host);
                }
            }

        };

        // Create a connection manager with custom configuration.
        PoolingHttpClientConnectionManager connManager = new PoolingHttpClientConnectionManager(
                socketFactoryRegistry, connFactory, dnsResolver);

        // Create socket configuration
        SocketConfig socketConfig = SocketConfig.custom()
            .setTcpNoDelay(true)
            .build();
        // Configure the connection manager to use socket configuration either
        // by default or for a specific host.
        connManager.setDefaultSocketConfig(socketConfig);
        connManager.setSocketConfig(new HttpHost("somehost", 80), socketConfig);
        // Validate connections after 1 sec of inactivity
        connManager.setValidateAfterInactivity(1000);

        // Create message constraints
        MessageConstraints messageConstraints = MessageConstraints.custom()
            .setMaxHeaderCount(200)
            .setMaxLineLength(2000)
            .build();
        // Create connection configuration
        ConnectionConfig connectionConfig = ConnectionConfig.custom()
            .setMalformedInputAction(CodingErrorAction.IGNORE)
            .setUnmappableInputAction(CodingErrorAction.IGNORE)
            .setCharset(Consts.UTF_8)
            .setMessageConstraints(messageConstraints)
            .build();
        // Configure the connection manager to use connection configuration either
        // by default or for a specific host.
        connManager.setDefaultConnectionConfig(connectionConfig);
        connManager.setConnectionConfig(new HttpHost("somehost", 80), ConnectionConfig.DEFAULT);

        // Configure total max or per route limits for persistent connections
        // that can be kept in the pool or leased by the connection manager.
        connManager.setMaxTotal(100);
        connManager.setDefaultMaxPerRoute(10);
        connManager.setMaxPerRoute(new HttpRoute(new HttpHost("somehost", 80)), 20);

        // Use custom cookie store if necessary.
        CookieStore cookieStore = new BasicCookieStore();
        // Use custom credentials provider if necessary.
        CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        // Create global request configuration
        RequestConfig defaultRequestConfig = RequestConfig.custom()
            .setCookieSpec(CookieSpecs.DEFAULT)
            .setExpectContinueEnabled(true)
            .setTargetPreferredAuthSchemes(Arrays.asList(AuthSchemes.NTLM, AuthSchemes.DIGEST))
            .setProxyPreferredAuthSchemes(Arrays.asList(AuthSchemes.BASIC))
            .build();

        // Create an HttpClient with the given custom dependencies and configuration.
        CloseableHttpClient httpclient = HttpClients.custom()
            .setConnectionManager(connManager)
            .setDefaultCookieStore(cookieStore)
            .setDefaultCredentialsProvider(credentialsProvider)
            .setProxy(new HttpHost("myproxy", 8080))
            .setDefaultRequestConfig(defaultRequestConfig)
            .build();

        try {
            HttpGet httpget = new HttpGet("http://httpbin.org/get");
            // Request configuration can be overridden at the request level.
            // They will take precedence over the one set at the client level.
            RequestConfig requestConfig = RequestConfig.copy(defaultRequestConfig)
                .setSocketTimeout(5000)
                .setConnectTimeout(5000)
                .setConnectionRequestTimeout(5000)
                .setProxy(new HttpHost("myotherproxy", 8080))
                .build();
            httpget.setConfig(requestConfig);

            // Execution context can be customized locally.
            HttpClientContext context = HttpClientContext.create();
            // Contextual attributes set the local context level will take
            // precedence over those set at the client level.
            context.setCookieStore(cookieStore);
            context.setCredentialsProvider(credentialsProvider);

            System.out.println("executing request " + httpget.getURI());
            CloseableHttpResponse response = httpclient.execute(httpget, context);
            try {
                System.out.println("----------------------------------------");
                System.out.println(response.getStatusLine());
                System.out.println(EntityUtils.toString(response.getEntity()));
                System.out.println("----------------------------------------");

                // Once the request has been executed the local context can
                // be used to examine updated state and various objects affected
                // by the request execution.

                // Last executed request
                context.getRequest();
                // Execution route
                context.getHttpRoute();
                // Target auth state
                context.getTargetAuthState();
                // Proxy auth state
                context.getTargetAuthState();
                // Cookie origin
                context.getCookieOrigin();
                // Cookie spec used
                context.getCookieSpec();
                // User security token
                context.getUserToken();

            } finally {
                response.close();
            }
        } finally {
            httpclient.close();
        }
    }

}
````

````
public class ApacheHttpClient {
    private static final Logger LOGGER = LoggerFactory.getLogger(ApacheHttpClient.class);

    private static final Integer DEFAULT_CONNECTION_TIMEOUT = 10000;
    private static final Integer DEFAULT_READ_TIMEOUT = 30000;

    private static CloseableHttpClient HTTP_ASYNC_CLIENT;

    static {
        SSLContext sslContext = null;
        try{
            //默认校验证书
            sslContext = SSLContexts.createSystemDefault();
        }
        catch (Exception ex){
            LOGGER.error("build http ssl context error",ex);
        }

        //检验域名
        HostnameVerifier hostnameVerifier = new DefaultHostnameVerifier();

        SocketConfig socketConfig = SocketConfig.custom()
                .setTcpNoDelay(true)
                .setSoKeepAlive(true)
                .setBacklogSize(1024)
                .setRcvBufSize(8092)
                .build();

        // Create a registry of custom connection socket factories for supported
        // protocol schemes.
        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
                .register("http", PlainConnectionSocketFactory.INSTANCE)
                .register("https", new SSLConnectionSocketFactory(sslContext,hostnameVerifier))
                .build();

        // Create a connection manager with custom configuration.
        PoolingHttpClientConnectionManager connManager = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
        connManager.setDefaultSocketConfig(socketConfig);
        // Validate connections after 1 sec of inactivity
        connManager.setValidateAfterInactivity(1000);

        // Configure total max or per route limits for persistent connections
        // that can be kept in the pool or leased by the connection manager.
        connManager.setMaxTotal(20480);
        connManager.setDefaultMaxPerRoute(1024);

        RequestConfig requestConfig = RequestConfig.custom()
                .setSocketTimeout(DEFAULT_READ_TIMEOUT)
                .setConnectTimeout(DEFAULT_CONNECTION_TIMEOUT)
                .setAuthenticationEnabled(false)
                .setRedirectsEnabled(false)
                .build();
        try {
            // Create an HttpClient with the given custom dependencies and configuration.
            HTTP_ASYNC_CLIENT = HttpClients.custom()
                    .setConnectionManager(connManager)
                    .setDefaultRequestConfig(requestConfig)
                    .build();
            LOGGER.info("========Http Client Start, class:{}==========",HTTP_ASYNC_CLIENT.getClass().getCanonicalName());
        } catch (Exception e) {
            LOGGER.error("===================init Http Client failed========================",e);
            Runtime.getRuntime().exit(0);
        }
    }


    public static byte[] execute(String url, String method, Map<String, String> header, byte[] body, Integer connectionTimeout, Integer readTimeOut) throws IOException {
        CloseableHttpResponse response = null;

        InputStream inputStream = null;

        try {
            if(!url.startsWith("http")){
                url = "http://"+url;
            }
            RequestConfig.Builder requestConfigBuilder = RequestConfig.custom();

            RequestBuilder requestBuilder = RequestBuilder.create(method.toUpperCase());

            //连接超时
            if(connectionTimeout != null){
                requestConfigBuilder.setConnectTimeout(connectionTimeout);
            }

            //读取超时
            if(readTimeOut != null){
                requestConfigBuilder.setSocketTimeout(readTimeOut).setConnectionRequestTimeout(readTimeOut);
            }

            if(header != null){
                //维持连接
                header.putIfAbsent("Connection", "keep-alive");

                //填充header
                for(Map.Entry<String, String> entry : header.entrySet()){
                    requestBuilder.addHeader(entry.getKey(),entry.getValue());
                }
            }

            //传输body
            if(body != null){
                HttpEntity httpEntity = new ByteArrayEntity(body);
                requestBuilder.setEntity(httpEntity);
            }

            requestBuilder.setUri(url);

            RequestConfig requestConfig = requestConfigBuilder.build();
            requestBuilder.setConfig(requestConfig);

            response = HTTP_ASYNC_CLIENT.execute(requestBuilder.build());

            if(response != null){
                HttpEntity entity = response.getEntity();
                if(entity != null && entity.isStreaming()) {
                    inputStream = entity.getContent();
                    if(inputStream != null) {
                        return IOUtils.toByteArray(inputStream);
                    }
                }
                else {
                    LOGGER.warn("response entity empty,url:{},code:{},reason:{}",
                            url,
                            response.getStatusLine().getStatusCode(),
                            response.getStatusLine().getReasonPhrase());
                }
            }

            return null;
        } catch (Exception e) {
            LOGGER.error("fetch http error,url:{}",url,e);
            throw e;
        }
        finally {
            if(inputStream != null){
                IOUtils.closeQuietly(inputStream);
            }
            if(response != null){
                try {
                    response.close();
                } catch (IOException e) {
                    LOGGER.error("close http response error,url:{}",url,e);
                }
            }
        }
    }
}
````


作者：李不言被占用了
链接：https://www.jianshu.com/p/f38a62efaa96
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
