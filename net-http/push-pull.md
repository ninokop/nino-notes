## ServerPush & ClientPull

前两天总结了一下HTTP2相关的发展过程，其中最重要的两点是：实现了服务端Push和多路复用。打算结合目前见过的一些在HTTP1.x上实现Push&Pull的实例，总结一下数据交互的实现方式：

1. etcd v2里的长轮询 long polling的方式
2. k8s-apiserver的stream的方式
3. service center和config center的websocket的方式
4. gRPC也就是http2的server push方式

### long polling

> 由于http1.x没有服务端push的方式，为了watch服务端的数据变化，最简单的办法当然是客户端去pull：客户端每隔定长时间去服务端拉数据同步，无论有没有服务端有没有数据变化。但是必然出现**通知不及时和大量无效的轮询。**long polling就是在这个polling的基础上的优化，当客户端发起long polling时，如果服务端没有相关数据，会hold住请求，直到服务端有数据要发或者超时才会返回。

#### client

etcdv2是个比较典型的long polling的例子。下面是客户端keysAPI的代码，它通过Watcher接口返回一个实现了Next方法的实例，客户端通过循环调用Next获得所有服务端事件。

```go
Watcher(key string, opts *WatcherOptions) Watcher
```

Next方法里client只是发了标记为wait的请求，通过统一的transport发到服务端。nexWait是用来生成请求体的，请求体的方法为GET，只是params带了wait字段，让服务端识别。

```go
func (hw *httpWatcher) Next(ctx context.Context) (*Response, error) {
	for {
		httpresp, body, err := hw.client.Do(ctx, &hw.nextWait)
        if err != nil {
			return nil, err
		}
		resp, err := unmarshalHTTPResponse(httpresp.StatusCode, httpresp.Header, body)
        if err != nil {
			if err == ErrEmptyBody {
				continue
			}
			return nil, err
		}
		hw.nextWait.WaitIndex = resp.Node.ModifiedIndex + 1
		return resp, nil
	}
}

```

#### server

> github.com/etcd/etcdserver/api/v2http/client.go

对应etcdv2的服务端keysHandler的处理过程是：调用etcdServer的Do方法，根据v2apistore的Get方法。如果请求中有wait字段，那么会返回一个kvStrore的watcher。

```go
func (h *keysHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	resp, err := h.server.Do(ctx, rr)
	switch {
	case resp.Watcher != nil:
		ctx, cancel := context.WithTimeout(context.Background(), defaultWatchTimeout)
		defer cancel()
		handleKeyWatch(ctx, w, resp, rr.Stream)
		...
	}
}
```

在处理watch请求时，通常都是使用context设置超时时间，但是这里defaultWatchTimeout设置的是maxInt64，所以watch的超时是客户端决定的，当超时发生close连接，server通过CloseNotifier得到通知并放弃处理。

> CloseNotifier Flusher

服务端首先把header flush到连接上，以免客户端等待header超时。之后等待内部kvstore的chan上有事件准备好，并发送。stream这个参数在etcdv2这个场景下为false，也就是long pollling获得数据即可以返回。

```go
func handleKeyWatch(ctx context.Context, w http.ResponseWriter, 
    resp etcdserver.Response, stream bool) {
	wa := resp.Watcher
	defer wa.Remove()
	ech := wa.EventChan()

	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("X-Etcd-Index", fmt.Sprint(wa.StartIndex()))
	w.Header().Set("X-Raft-Index", fmt.Sprint(resp.Index))
	w.Header().Set("X-Raft-Term", fmt.Sprint(resp.Term))
	w.WriteHeader(http.StatusOK)
	// Ensure headers are flushed early, in case of long polling
	w.(http.Flusher).Flush()
	for {
		select {
		case <-nch: // CloseNotifier, Client closed connection. Nothing to do.
			return
		case <-ctx.Done(): // Timed out.
			return
		case ev, ok := <-ech:
			if !ok {
				return
			}
			ev = trimEventPrefix(ev, etcdserver.StoreKeysPrefix)
			if err := json.NewEncoder(w).Encode(ev); err != nil {
				plog.Warningf("error writing event (%v)", err)
				return
			}
			if !stream {
				return
			}
			w.(http.Flusher).Flush()
		}
	}
}
```

### streaming

stream是要在同一个连接上，分多个部分发送HTTP响应，通常都是用Chunked编码。就是服务端把chunk data在同一个连接上发给客户端。一般HTTP的响应中发送的数据是整个发送，并且通过Content-Length消息头字段表示数据的长度。引入分块传输编码，可以使HTTP服务端动态生成内容，消息体由数量未定的块组成，并且以最后一个大小为0的块结束。

```powershell
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

25
This is the data in the first chunk

1C
and this is the second one

3
con

8 
sequence

0

```

#### server & serveWatch

k8s的服务端watch接口是通过etcd的watch接口实现的长连接方式。最终注册到go-restful的Watch路由，对应GET方法和ListResource这个handlerFunc。

> k8s.io/apiserver/pkg/endpoints

```go
func ListResource(r rest.Lister, rw rest.Watcher, scope RequestScope, 
     forceWatch bool, minRequestTimeout time.Duration) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		...
		if opts.Watch || forceWatch {
			if rw == nil {
				return
			}
			timeout := time.Duration(0)
			if opts.TimeoutSeconds != nil {
				timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
			}
			watcher, err := rw.Watch(ctx, &opts)
			if err != nil {
				scope.err(err, w, req)
				return
			}
			serveWatch(watcher, scope, req, w, timeout)
			return
}}}

```

其中watcher是内部storage通过etcd的watch接口封装的返回事件的chan。serveWatch就是在处理这个内部chan，并把chan上发生的事件通过chunk编码发给客户端。这个循环可能因为客户端close连接或超时而结束。

```go
func serveWatch(watcher watch.Interface, scope RequestScope, 
    req *http.Request, w http.ResponseWriter, timeout time.Duration) {
	// negotiate for the stream serializer ...
	server := &WatchServer{
		Watching: watcher,
		Scope:    scope,
		UseTextFraming:  useTextFraming,
		MediaType:       mediaType + ";stream=watch,
		Framer:          framer,
		Encoder:         encoder,
		EmbeddedEncoder: embeddedEncoder,
        Fixup: func(obj runtime.Object) {},
		TimeoutFactory: &realTimeoutFactory{timeout},
	}
	server.ServeHTTP(w, req)
}

func (s *WatchServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // if isWebSocketRequest
	cn, ok1 := w.(http.CloseNotifier)
	flusher, ok2 := w.(http.Flusher)
	if !ok1 || !ok2 {
		return
	}

	framer := s.Framer.NewFrameWriter(w)
	e := streaming.NewEncoder(framer, s.Encoder)
	timeoutCh, cleanup := s.TimeoutFactory.TimeoutCh()

	w.Header().Set("Content-Type", s.MediaType)
	w.Header().Set("Transfer-Encoding", "chunked")
	w.WriteHeader(http.StatusOK)
	flusher.Flush()

	buf := &bytes.Buffer{}
	ch := s.Watching.ResultChan()
	for {
		select {
		case <-cn.CloseNotify():
			return
		case <-timeoutCh:
			return
		case event, ok := <-ch:
			if !ok {
				return
			}
			obj := event.Object
			s.EmbeddedEncoder.Encode(obj, buf)

			unknown.Raw = buf.Bytes()
			event.Object = &unknown
			metav1.Convert_versioned_InternalEvent_to_versioned_Event(
               metav1.InternalEvent(event), 
               &metav1.WatchEvent{}, nil)
          
			e.Encode(outEvent)
			if len(ch) == 0 {
				flusher.Flush()
			}
			buf.Reset()
		}
	}
}

```

#### registry&storage

> 这一节本来跟stream没有关系，但它是对etcd的watch的封装所以还是记一下。

上面内部watcher是rest.StandardStorage接口，它是以下所有接口的组合。它的实现`genericregistry.Store`提供了N个函数挂载点，对所有资源类型提供了统一的实现。比如每种资源都实现了NewFunc和KeyFunc，Store统一实现Creater接口实现对每种资源的创建，并最终调用storage包面向etcd的接口实现到后端数据库的持久化。

```go
type StandardStorage interface {
	Getter
	Lister
	CreaterUpdater
	GracefulDeleter
	CollectionDeleter
	Watcher
}
```

比如store封装的watch接口最终到storage里面向etcd的watch接口。

```go
func (e *Store) Watch(ctx context.Context, 
    options *metainternalversion.ListOptions) (watch.Interface, error) {
	predicate := e.PredicateFunc(labels.Everything(), fields.Everything())
	return e.WatchPredicate(ctx, predicate, options.ResourceVersion)
}
func (e *Store) WatchPredicate(ctx context.Context, 
    p storage.SelectionPredicate, resourceVersion string)
    (watch.Interface, error) {
    ...
	w, err := e.Storage.WatchList(ctx, e.KeyRootFunc(ctx), resourceVersion, p)
	return w, nil
}
```

etcdHelper这个包封装了etcdv2的接口，最终是通过循环处理Watcher.Next来实现内部事件的产生。这个过程还涉及到storage的watch cache。详细过程下一次写watch cache再写吧。

```go
func (h *etcdHelper) WatchList(ctx context.Context, key string, 
resourceVersion string, pred storage.SelectionPredicate) (watch.Interface, error) {
	key = path.Join(h.pathPrefix, key)
	w := newEtcdWatcher(true, h.quorum, exceptKey(key), pred, 
         h.codec, h.versioner, nil, h.transformer, h)
	go w.etcdWatch(ctx, h.etcdKeysAPI, key, resourceVersion)
	return w, nil
}
```

```go
func (w *etcdWatcher) etcdWatch(ctx context.Context, client etcd.KeysAPI, 
    key string, resourceVersion uint64) {
	var watcher etcd.Watcher
	returned := func() bool {
        ...
		opts := etcd.WatcherOptions{
			Recursive:  w.list,
			AfterIndex: resourceVersion,
		}
		watcher = client.Watcher(key, &opts)
		w.ctx, w.cancel = context.WithCancel(ctx)
		return false
	}()

	if returned {
		return
	}
	for {
		resp, err := watcher.Next(w.ctx)
		w.etcdIncoming <- resp
	}
}
```

#### client

客户端通过Do获取到服务端的第一个Header响应。最后通过StreamWatcher封装好watch的ResultChan接口，它从连接上decoder反序列化数据由streamWatcher封装好返回。

```go
func (r *Request) Watch() (watch.Interface, error) {
	url := r.URL().String()
	req, err := http.NewRequest(r.verb, url, r.body)
	req.Header = r.headers
	client := r.client
	
    resp, err := client.Do(req)
	if resp.StatusCode != http.StatusOK {
		defer resp.Body.Close()
		return nil, fmt.Errorf("got status: %v", resp.StatusCode)
	}
	framer := r.serializers.Framer.NewFrameReader(resp.Body)
	decoder := streaming.NewDecoder(framer, r.serializers.StreamingSerializer)
	return watch.NewStreamWatcher(
      restclientwatch.NewDecoder(decoder, r.serializers.Decoder)), nil
}
```

这个接口最常见的地方是在reflector的listWatch当中， 在watch循环中通常客户端会指定超时时间，5分钟，好让服务端知道什么时候超时结束。

```go
for {
	select {
	case <-stopCh:
		return nil
	default:
	}

	timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
	options = metav1.ListOptions{
		ResourceVersion: resourceVersion,
		TimeoutSeconds: &timeoutSeconds,
	}

	w, err := r.listerWatcher.Watch(options)
	if err != nil {
		switch err {
		case io.EOF:// watch closed normally
		case io.ErrUnexpectedEOF:
		default:
		}
		return nil
	}
	r.watchHandler(w, &resourceVersion, resyncerrc, stopCh)
}
```

### WebSocket



























