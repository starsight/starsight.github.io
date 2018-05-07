---
layout: post
title: "Volley理解"
date: 2018-05-07 11:17:12 +0800
comments: true
categories: 2018-05
tags: [android,volley]
---
### 概述

**物理质量**

- 使用Volley 需要Volley.jar(120k)，加上自己的封装最多140k。
- 使用OkHttp需要 okio.jar (80k), okhttp.jar(330k)这2个jar包，总大小差不多400k,加上自己的封装，差不多得410k。

**Volley 的优点**

- 非常适合进行数据量不大，但通信频繁的网络操作
- 可直接在主线程调用服务端并处理返回结果
- 可以取消请求，容易扩展，面向接口编程<!--more-->
- 网络请求线程NetworkDispatcher默认开启了4个，可以优化，通过手机CPU数量
- 通过使用标准的HTTP缓存机制保持磁盘和内存响应的一致

**Volley 的缺点**

- 使用的是httpclient、HttpURLConnection
- 6.0不支持httpclient了，如果想支持得添加org.apache.http.legacy.jar
- 对大文件下载 Volley的表现非常糟糕
- 只支持http请求
- 图片加载性能一般

![](/images/android/volley/Volley-architecture.png)

对于这张图我们只需要知道：

- Volley 运行的过程中一共有三种线程，包括 UI 线程、Cache 调度线程和 NetWork 调度线程池
- 请求加入优先级队列，Cache 线程进行筛选，如果命中（hit）分发给 UI 线程
- 未命中（miss）交给 NetWork 调度线程池处理，取回后更新 Cache 并分发给 UI 线程
- 每次请求执行过程始于 UI 线程， 终于 UI 线程

![](/images/android/volley/Volley.png)

RequestQueue封装了众多重要参数，重点有缓存队列，网络请求队列，cache，network，缓存调度，网络调度数组。

流程全部在图中。
- 当一个RequestQueue被成功申请后会开启一个CacheDispatcher和4个默认的NetworkDispatcher。
- CacheDispatcher缓存调度器最为第一层缓冲，开始工作后阻塞的从缓存序列mCacheQueue中取得请求；对于已经取消的请求，标记为跳过并结束这个请求；新的或者过期的请求，直接放入mNetworkQueue中由N个NetworkDispatcher进行处理；已获得缓存信息（网络应答）却没有过期的请求，由Request的parseNetworkResponse进行解析，从而确定此应答是否成功。然后将请求和应答交由Delivery分发者进行处理，如果需要更新缓存那么该请求还会被放入mNetworkQueue中。
- 将请求Request add到RequestQueue后对于不需要缓存的请求（需要额外设置，默认是需要缓存）直接丢入mNetworkQueue交给N个NetworkDispatcher处理；对于需要缓存的，新的请求加到mCacheQueue中给CacheDispatcher处理；需要缓存，但是缓存列表中已经存在了相同URL的请求，放在mWaitingQueue中做暂时处理，等待之前请求完毕后，再重新添加到mCacheQueue中。
- 网络请求调度器NetworkDispatcher作为网络请求真实发生的地方，对消息交给BasicNetwork进行处理，同样的，请求和结果都交由Delivery分发者进行处理。
- Delivery分发者实际上已经是对网络请求处理的最后一层了，在Delivery对请求处理之前，Request已经对网络应答进行过解析，此时应答成功与否已经设定；而后Delivery根据请求所获得的应答情况做不同处理；若应答成功，则触发deliverResponse方法，最终会触发开发者为Request设定的Listener；若应答失败，则触发deliverError方法，最终会触发开发者为Request设定的ErrorListener；处理完后，一个Request的生命周期就结束了，Delivery会调用Request的finish操作，将其从mRequestQueue中移除，与此同时，如果等待列表中存在相同URL的请求，则会将剩余的层级请求全部丢入mCacheQueue交由CacheDispatcher进行处理。

### 缓存类

Volley的缓存，主要是通过DiskBasedCache实现的。DiskBasedCache类，继承于Cache接口。DiskBasedCache保存CacheHeader，具体的数据并不会全部缓存到内存中（只有一个CacheHeader）。

在初始化时，会先把存在的cache（CacheHeader）保存进mEntries。

可以从get方法看出，先从mEntries找到CacheHeader，再去读文件，根据entry.toCacheEntry(data);构建出Entry类对象。

```java
public class DiskBasedCache implements Cache {

    /** Map of the Key, CacheHeader pairs */
    private final Map<String, CacheHeader> mEntries =
            new LinkedHashMap<String, CacheHeader>(16, .75f, true);

    /** Total amount of space currently used by the cache in bytes. */
    private long mTotalSize = 0;

    /** The root directory to use for the cache. */
    private final File mRootDirectory;

    /** The maximum size of the cache in bytes. */
    private final int mMaxCacheSizeInBytes;
    
    ......
        
    public synchronized void initialize() {
        //初始化，会读取缓存目录下的文件，全部放入mEntries中
        ...
        File[] files = mRootDirectory.listFiles();
        ...
        for (File file : files) {
            BufferedInputStream fis = null;
            try {
                fis = new BufferedInputStream(new FileInputStream(file));
                CacheHeader entry = CacheHeader.readHeader(fis);
                entry.size = file.length();
                putEntry(entry.key, entry);//重点
            } catch (IOException e) {
                ...
            } finally {
                ...
            }
        }
    }
    @Override
    public synchronized Entry get(String key) {
        CacheHeader entry = mEntries.get(key);
        // if the entry does not exist, return.
        if (entry == null) {
            return null;
        }

        File file = getFileForKey(key);
        CountingInputStream cis = null;
        try {
            cis = new CountingInputStream(new FileInputStream(file));
            CacheHeader.readHeader(cis); // eat header  //先读header
            byte[] data = streamToBytes(cis, (int) (file.length() - cis.bytesRead));
            return entry.toCacheEntry(data);//重点
        } catch (IOException e) {
            ...
        } finally {
            ...
        }
    }
        
    static class CacheHeader{见下代码}
    private static class CountingInputStream extends FilterInputStream {...}
}

//DiskBasedCache内部类，提出来分析 注意没有data信息
static class CacheHeader {
        /** The size of the data identified by this CacheHeader. (This is not
         * serialized to disk. */
        public long size;

        /** The key that identifies the cache entry. */
        public String key;

        /** ETag for cache coherence. */
        public String etag;

        /** Date of this response as reported by the server. */
        public long serverDate;

        /** The last modified date for the requested object. */
        public long lastModified;

        /** TTL for this record. */
        public long ttl;

        /** Soft TTL for this record. */
        public long softTtl;

        /** Headers from the response resulting in this cache entry. */
        public Map<String, String> responseHeaders;

        private CacheHeader() { }

        public CacheHeader(String key, Entry entry) {
            this.key = key;
            this.size = entry.data.length;
            this.etag = entry.etag;
            this.serverDate = entry.serverDate;
            this.lastModified = entry.lastModified;
            this.ttl = entry.ttl;
            this.softTtl = entry.softTtl;
            this.responseHeaders = entry.responseHeaders;
        }
    
	public static CacheHeader readHeader(InputStream is) throws IOException{...}
	public boolean writeHeader(OutputStream os){...}
	public Entry toCacheEntry(byte[] data){
            Entry e = new Entry();
            e.data = data;
            e.etag = etag;
            e.serverDate = serverDate;
            e.lastModified = lastModified;
            e.ttl = ttl;
            e.softTtl = softTtl;
            e.responseHeaders = responseHeaders;
            return e;
    }
}
      
```

Cache接口及静态内部类Entry

```java
public interface Cache {
    public Entry get(String key);
    public void put(String key, Entry entry);
    public void initialize();
    public void invalidate(String key, boolean fullExpire);
    public void remove(String key);
    public void clear();

    public static class Entry {
        /** The data returned from cache. */
        public byte[] data;

        /** ETag for cache coherency. */
        public String etag;

        /** Date of this response as reported by the server. */
        public long serverDate;

        /** The last modified date for the requested object. */
        public long lastModified;

        /** TTL for this record. */
        public long ttl;

        /** Soft TTL for this record. */
        public long softTtl;

        /** Immutable response headers as received from server; must be non-null. */
        public Map<String, String> responseHeaders = Collections.emptyMap();

        /** True if the entry is expired. */
        public boolean isExpired() {
            return this.ttl < System.currentTimeMillis();
        }

        /** True if a refresh is needed from the original data source. */
        public boolean refreshNeeded() {
            return this.softTtl < System.currentTimeMillis();
        }
    }
}
```


有人说Volley的缓存替换策略是FIFO的，不是LRU。我看了下源码，似乎不是这样（如果我弄错了，希望各位指正）

FIFO 说法来源  [Volley HTTP 缓存机制](https://blog.csdn.net/wzy_1988/article/details/51540668)    [手撕 Volley（二）](https://www.jianshu.com/p/358b766c8d27)

DiskBasedCache中的相关属性方法

```java
private final Map<String, CacheHeader> mEntries =
            new LinkedHashMap<String, CacheHeader>(16, .75f, true);
//重点，这个true是设置成access order， false是insertion order

/**
 * Puts the entry with the specified key into the cache.
 */
@Override
public synchronized void put(String key, Entry entry) {
    pruneIfNeeded(entry.data.length);
    File file = getFileForKey(key);
    try {
        BufferedOutputStream fos = new BufferedOutputStream(new FileOutputStream(file));
        CacheHeader e = new CacheHeader(key, entry);
        boolean success = e.writeHeader(fos);
        if (!success) {
            fos.close();
            VolleyLog.d("Failed to write header for %s", file.getAbsolutePath());
            throw new IOException();
        }
        fos.write(entry.data);
        fos.close();
        putEntry(key, e);
        return;
    } catch (IOException e) {
    }
    boolean deleted = file.delete();
    if (!deleted) {
        VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
    }
}

    /**
     * Prunes the cache to fit the amount of bytes specified.
     * @param neededSpace The amount of bytes we are trying to fit into the cache.
     */
    private void pruneIfNeeded(int neededSpace) {
        if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes) {
            return;
        }
        if (VolleyLog.DEBUG) {
            VolleyLog.v("Pruning old cache entries.");
        }

        long before = mTotalSize;
        int prunedFiles = 0;
        long startTime = SystemClock.elapsedRealtime();

        Iterator<Map.Entry<String, CacheHeader>> iterator = mEntries.entrySet().iterator();
        while (iterator.hasNext()) {
            //从头开始删除，linkedhashmap又是access order ,所以是LRU
            Map.Entry<String, CacheHeader> entry = iterator.next();
            CacheHeader e = entry.getValue();
            boolean deleted = getFileForKey(e.key).delete();
            if (deleted) {
                mTotalSize -= e.size;
            } else {
               VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",
                       e.key, getFilenameForKey(e.key));
            }
            iterator.remove();
            prunedFiles++;

            if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes * HYSTERESIS_FACTOR) {
                break;
            }
        }

        if (VolleyLog.DEBUG) {
            VolleyLog.v("pruned %d files, %d bytes, %d ms",
                    prunedFiles, (mTotalSize - before), SystemClock.elapsedRealtime() - startTime);
        }
    }
```


### 缓存添加

缓存的添加逻辑在NetworkDispatcher.java中展开

```java
//NetworkDispatcher.java
public class NetworkDispatcher extends Thread {
    /** The queue of requests to service. */
    private final BlockingQueue<Request<?>> mQueue;
    /** The network interface for processing requests. */
    private final Network mNetwork;
    /** The cache to write to. */
    private final Cache mCache;
    /** For posting responses and errors. */
    private final ResponseDelivery mDelivery;
    /** Used for telling us to die. */
    private volatile boolean mQuit = false;
    
    ...
        
    @Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            Request<?> request;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch ...

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                // Perform the network request.
                NetworkResponse networkResponse = mNetwork.performRequest(request);//发起请求
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);//转1 目的在于构建Cache.Entry
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);//放入mCache中（cacheHeader），同时保存在磁盘中（这个方法在上文FIFO/LRU讨论有提及）
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch ...
        }
    }
    
    //其他方法
}

//Request.java
//接 转1
public abstract class Request<T> implements Comparable<Request<T>> {
    ...
 	//在具体的Request类中实现，这里以JsonObjectRequest类为例，见下代码
	abstract protected Response<T> parseNetworkResponse(NetworkResponse response);
    ...
}

//JsonObjectRequest.java 以JsonObjectRequest的实现为例
public class JsonObjectRequest extends JsonRequest<JSONObject> {
    public JsonObjectRequest(int method, String url, JSONObject jsonRequest,
            Listener<JSONObject> listener, ErrorListener errorListener) {
        super(method, url, (jsonRequest == null) ? null : jsonRequest.toString(), listener,
                    errorListener);
    }
    ...
    //实现抽象方法
    @Override
    protected Response<JSONObject> parseNetworkResponse(NetworkResponse response) {
        try {
            String jsonString = new String(response.data,
                    HttpHeaderParser.parseCharset(response.headers, PROTOCOL_CHARSET));
            // HttpHeaderParser.parseCacheHeaders() -> 完成response的header解析,构建成Cache.Entry（这是有数据的）
            // Response.success() -> new Response<T>(T result, Cache.Entry cacheEntry); 
            return Response.success(new JSONObject(jsonString),
                    HttpHeaderParser.parseCacheHeaders(response));//静态方法
        } catch (UnsupportedEncodingException e) {
            return Response.error(new ParseError(e));
        } catch (JSONException je) {
            return Response.error(new ParseError(je));
        }
    }
}

//HttpHeaderParser.java
public class HttpHeaderParser {
    public static Cache.Entry parseCacheHeaders(NetworkResponse response) {
        long now = System.currentTimeMillis();

        Map<String, String> headers = response.headers;

        long serverDate = 0;
        long lastModified = 0;
        long serverExpires = 0;
        long softExpire = 0;
        long finalExpire = 0;
        long maxAge = 0;
        long staleWhileRevalidate = 0;
        boolean hasCacheControl = false;
        boolean mustRevalidate = false;

        String serverEtag = null;
        String headerValue;

        headerValue = headers.get("Date");
        if (headerValue != null) {
            serverDate = parseDateAsEpoch(headerValue);
        }

        headerValue = headers.get("Cache-Control");
        if (headerValue != null) {
            hasCacheControl = true;
            String[] tokens = headerValue.split(",");
            for (int i = 0; i < tokens.length; i++) {
                String token = tokens[i].trim();
                if (token.equals("no-cache") || token.equals("no-store")) {
                    return null;
                } else if (token.startsWith("max-age=")) {
                    try {
                        maxAge = Long.parseLong(token.substring(8));
                    } catch (Exception e) {
                    }
                } else if (token.startsWith("stale-while-revalidate=")) {
                    try {
                        staleWhileRevalidate = Long.parseLong(token.substring(23));
                    } catch (Exception e) {
                    }
                } else if (token.equals("must-revalidate") || token.equals("proxy-revalidate")) {
                    mustRevalidate = true;
                }
            }
        }

        headerValue = headers.get("Expires");
        if (headerValue != null) {
            serverExpires = parseDateAsEpoch(headerValue);
        }

        headerValue = headers.get("Last-Modified");
        if (headerValue != null) {
            lastModified = parseDateAsEpoch(headerValue);
        }

        serverEtag = headers.get("ETag");

        // Cache-Control takes precedence over an Expires header, even if both exist and Expires
        // is more restrictive.
        if (hasCacheControl) {
            softExpire = now + maxAge * 1000;
            finalExpire = mustRevalidate
                    ? softExpire
                    : softExpire + staleWhileRevalidate * 1000;
        } else if (serverDate > 0 && serverExpires >= serverDate) {
            // Default semantic for Expire header in HTTP specification is softExpire.
            softExpire = now + (serverExpires - serverDate);
            finalExpire = softExpire;
        }

        Cache.Entry entry = new Cache.Entry();
        entry.data = response.data;
        entry.etag = serverEtag;
        entry.softTtl = softExpire;
        entry.ttl = finalExpire;
        entry.serverDate = serverDate;
        entry.lastModified = lastModified;
        entry.responseHeaders = headers;

        return entry;
    }
    
	...
}
```



Volley对于图片加载做了内存缓存的处理(ImageLoader)，详情见[Volley图片加载功能](https://blog.csdn.net/wzy_1988/article/details/51005389)

### 优秀的参考链接

[手撕Volley（一、二、三）](https://www.jianshu.com/p/33be82da8f25)
[Volley HTTP 缓存机制](https://blog.csdn.net/wzy_1988/article/details/51540668)
[Volley图片加载功能](https://blog.csdn.net/wzy_1988/article/details/51005389)
[彻底弄懂HTTP缓存机制及原理](http://www.cnblogs.com/chenqf/p/6386163.html)
[浅谈浏览器http的缓存机制](https://www.cnblogs.com/vajoy/p/5341664.html)