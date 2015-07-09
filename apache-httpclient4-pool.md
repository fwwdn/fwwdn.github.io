title: HttpClient4.0 Http连接池
date: 2015-06-14 21:26:47
tags: 后端技术
---

连接池技术作为创建和管理连接的缓冲池技术，目前已广泛用于诸如数据库连接等长连接的维护和管理中，能够有效减少系统的响应时间，节省服务器资源开销。其优势主要有两个：其一是减少创建连接的资源开销，其二是资源的访问控制。连接池管理的对象是长连接，对于HTTP连接是否适用，我们需要首先回顾一下长连接和短连接。

所谓长连接是指客户端与服务器端一旦建立连接以后，可以进行多次数据传输而不需重新建立连接，而短连接则每次数据传输都需要客户端和服务器端建立一次连接。长连接的优势在于省去了每次数据传输连接建立的时间开销，能够大幅度提高数据传输的速度，对于P2P应用十分适合，但是对于诸如Web网站之类的B2C应用，并发请求量大，每一个用户又不需频繁的操作的场景下，维护大量的长连接对服务器无疑是一个巨大的考验。而此时，短连接可能更加适用。但是短连接每次数据传输都需要建立连接，我们知道HTTP协议的传输层协议是TCP协议，TCP连接的建立和释放分别需要进行3次握手和4次握手，频繁的建立连接即增加了时间开销，同时频繁的创建和销毁Socket同样是对服务器端资源的浪费。所以对于需要频繁发送HTTP请求的应用，需要在客户端使用HTTP长连接。

HTTP连接是无状态的，这样很容易给我们造成HTTP连接是短连接的错觉，实际上HTTP1.1默认即是持久连接，HTTP1.0也可以通过在请求头中设置Connection:keep-alive使得连接为长连接。既然HTTP协议支持长连接，我们就有理由相信HTTP连接同样需要连接池技术来管理和维护连接建立和销毁。HTTP Client4.0的ThreadSafeClientConnManager实现了HTTP连接的池化管理，其管理连接的基本单位是Route（路由），每个路由上都会维护一定数量的HTTP连接。这里的Route的概念可以理解为客户端机器到目标机器的一条线路，例如使用HttpClient的实现来分别请求 www.163.com 的资源和 www.sina.com 的资源就会产生两个route。缺省条件下对于每个Route，HttpClient仅维护2个连接，总数不超过20个连接，显然对于大多数应用来讲，都是不够用的，可以通过设置HTTP参数进行调整。

    HttpParams params = new BasicHttpParams();
	//将每个路由的最大连接数增加到200
	ConnManagerParams.setMaxTotalConnections(params,200);
	// 将每个路由的默认连接数设置为20
	ConnPerRouteBean connPerRoute = new ConnPerRouteBean(20);
	// 设置某一个IP的最大连接数
	HttpHost localhost = new HttpHost("locahost", 80);
	connPerRoute.setMaxForRoute(new HttpRoute(localhost), 50);
	ConnManagerParams.setMaxConnectionsPerRoute(params, connPerRoute);
	SchemeRegistry schemeRegistry = new SchemeRegistry();
	schemeRegistry.register(
	new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
	schemeRegistry.register(
	new Scheme("https", SSLSocketFactory.getSocketFactory(), 443));
	ClientConnectionManager cm = new ThreadSafeClientConnManager(params, schemeRegistry);
	HttpClient httpClient = new DefaultHttpClient(cm, params);

可以配置的HTTP参数有：

 1、 http.conn-manager.timeout 当某一线程向连接池请求分配线程时，如果连接池已经没有可以分配的连接时，该线程将会被阻塞，直至http.conn-manager.timeout超时，抛出ConnectionPoolTimeoutException。

 2、 http.conn-manager.max-per-route 每个路由的最大连接数； 

 3、 http.conn-manager.max-total 总的连接数；

连接的有效性检测是所有连接池都面临的一个通用问题，大部分HTTP服务器为了控制资源开销，并不会永久的维护一个长连接，而是一段时间就会关闭该连接。放回连接池的连接，如果在服务器端已经关闭，客户端是无法检测到这个状态变化而及时的关闭Socket的。这就造成了线程从连接池中获取的连接不一定是有效的。这个问题的一个解决方法就是在每次请求之前检查该连接是否已经存在了过长时间，可能已过期。但是这个方法会使得每次请求都增加额外的开销。HTTP Client4.0的ThreadSafeClientConnManager 提供了
closeExpiredConnections()方法和closeIdleConnections()方法来解决该问题。前一个方法是清除连接池中所有过期的连接，至于连接什么时候过期可以设置，设置方法将在下面提到，而后一个方法则是关闭一定时间空闲的连接，可以使用一个单独的线程完成这个工作。

    public static class IdleConnectionMonitorThread extends Thread {
		private final ClientConnectionManager connMgr;
		private volatile boolean shutdown;
		public IdleConnectionMonitorThread(ClientConnectionManager connMgr) {
			super();
			this.connMgr = connMgr;
		}
		@Override
		public void run() {
			try {
				while (!shutdown) {
					synchronized (this) {
						wait(5000);
						// 关闭过期的连接
						connMgr.closeExpiredConnections();
						// 关闭空闲时间超过30秒的连接
						connMgr.closeIdleConnections(30, TimeUnit.SECONDS);
					}
				}
			} catch (InterruptedException ex) {
				// terminate
			}
		}
		public void shutdown() {
			shutdown = true;
			synchronized (this) {
			notifyAll();
		}
	}

刚才提到，客户端可以设置连接的过期时间，可以通过HttpClient的setKeepAliveStrategy方法设置连接的过期时间，这样就可以配合closeExpiredConnections()方法解决连接池中连接失效的。

	DefaultHttpClient httpclient = new DefaultHttpClient();
	httpclient.setKeepAliveStrategy(new ConnectionKeepAliveStrategy() {
		public long getKeepAliveDuration(HttpResponse response, HttpContext context) {
			// Honor 'keep-alive' header
			HeaderElementIterator it = new BasicHeaderElementIterator(
			response.headerIterator(HTTP.CONN_KEEP_ALIVE));
			while (it.hasNext()) {
				HeaderElement he = it.nextElement();
				String param = he.getName();
				String value = he.getValue();
				if (value != null && param.equalsIgnoreCase("timeout")) {
					try {
						return Long.parseLong(value) * 1000;
					} catch(NumberFormatException ignore) {
					}
				}
			}
		HttpHost target = (HttpHost) context.getAttribute(
		ExecutionContext.HTTP_TARGET_HOST);
		if ("www.qq.com".equalsIgnoreCase(target.getHostName())) {
			// 对于qq这个路由的连接，保持5秒
			return 5 * 1000;
		} else {
			// 其他路由保持30秒
			return 30 * 1000;
		}
	}
	});