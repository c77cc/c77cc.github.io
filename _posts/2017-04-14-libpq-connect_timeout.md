---
layout: post
title: github.com/lib/pq connect_timeout的一个问题
---

前两天项目报了了个BUG，它占着PG几十条链接一直不释放，导致控制台打不开；

咱们的exe都是有MaxIdleConn和MaxOpenConn控制的，如果不是看了pg_stat_activity，我是不太相信的，但事实却打了我一巴掌

查看出问题的环境的日志，发现日志里每出现一次与PG通讯的超时(**tcp read timeout**)，就会占用一条链接不释放

在连接PG的参数里，有个叫做**connect_timeout**的东西，很有可能就是这货啊。。。

**connect_timeout** 这参数在 **github.com/lib/pq**这个pkg里边，来看看它是怎么用的吧

```golang
func dial(d Dialer, o values) (net.Conn, error) {
	ntw, addr := network(o)
	// SSL is not necessary or supported over UNIX domain sockets
	if ntw == "unix" {
		o["sslmode"] = "disable"
	}

	// Zero or not specified means wait indefinitely.
	if timeout := o.Get("connect_timeout"); timeout != "" && timeout != "0" {
		seconds, err := strconv.ParseInt(timeout, 10, 0)
		if err != nil {
			return nil, fmt.Errorf("invalid value for parameter connect_timeout: %s", err)
		}
		duration := time.Duration(seconds) * time.Second
		deadline := time.Now().Add(duration)
		conn, err := d.DialTimeout(ntw, addr, duration)
		if err != nil {
			return nil, err
		}
		err = conn.SetDeadline(deadline)
		return conn, err
	}
	return d.Dial(ntw, addr)
}
```

看起来好像木有什么问题啊，

- d.DialTimeout() 创建TCP链接用到了connect_timeout
- conn.SetDeadline() 也用到了connect_timeout，好奇怪这个参数这怎么也用了捏

去翻了下这个库在github的pr，发现connect_timeout是在这<https://github.com/lib/pq/pull/229>加上的，SetDeadline是哥们后来再次加上的，猜测他的意思是connect_timeout不仅仅是应该在创建链接时生效，应该在链接的整个生命周期都生效，好吧，这样看起来也没啥问题。

那么问题来了，当发生timeout时，database/sql难道没有释放关闭链接么，上代码(Golang 1.7.4 line-822)

```golang
	db.numOpen++ // optimistically
	db.mu.Unlock()
	ci, err := db.driver.Open(db.dsn)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
```

嗯，当创建链接时出错，不会关闭释放链接（链接都没创建成功，不需要释放，这里是正确的），而且还会 **db.maybeOpenNewConnections**尝试打开一条新链接，有疑点，但是问题不出在这- -，因为db.connRequests为0，这里不具体讨论这个函数了，好奇的你可以去看database/sql/sql.go line-710。


好了，问题确定了：**发生超时，链接确实没有被释放**

理下思路先，我们这下把链接分为2个阶段，

- TCP链接，实际的TCP链接
- database/sql 里抽象的Conn，可以理解为PG链接
- 按照顺序来，应该先有TCP链接，才有PG链接

那么超时是发生在哪个阶段的链接呢

- 如果超时发生在TCP链接阶段，那么并不需要释放链接
- 如果超时发生在PG链接阶段，那么就需要释放TCP链接

很明显，超时肯定发生在PG链接阶段，那么转移去看 **lib/pq/conn.go**，当TCP链接创建成功后，与PG是怎么通讯的呢？

lib/pq/conn.go line-206

```golang
	c, err := dial(d, o)
	if err != nil {
		return nil, err
	}

	cn := &conn{c: c}
	cn.ssl(o)
	cn.buf = bufio.NewReader(cn.c)
	cn.startup(o)
	// reset the deadline, in case one was set (see dial)
	if timeout := o.Get("connect_timeout"); timeout != "" && timeout != "0" {
		err = cn.c.SetDeadline(time.Time{})
	}
	return cn, err

```

原来在cn.startup里边，接着转移

```golang
	for {
		t, r := cn.recv()
		switch t {
		case 'K':
		case 'S':
			cn.processParameterStatus(r)
		...

```

```golang
func (cn *conn) recv() (t byte, r *readBuf) {
	for {
		var err error
		r = &readBuf{}
		t, err = cn.recvMessage(r)
		if err != nil {
			panic(err)
		}

		switch t {
		case 'E':
			panic(parseError(r))
		case 'N':
			// ignore
		default:
			return
		}
	}
}

```

还记得前面的**conn.SetDeadline**么，如果设置了这个超时时间，那么recv里边肯定会超时的，一旦超时，抛出的异常会在上层被捕获，最后回到

```golang
    ci, err := db.driver.Open(db.dsn)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
```

问题找到了，db.numOpen--，这也解释了即使设置了MaxOpenConn也会无效，lib/pq/conn.go connect_timeout功能的作者给自己、用户抛了个坑。



重新梳理下问题的流程

1. 创建TCP链接成功
2. 与PG通信过程时，发生超时
3. database/sql认为创建链接失败，无需关闭链接，直接返回
4. 程序员们 db.Query 返回错误，无需关闭链接，打印错误，函数返回
5. 结果：TCP链接没释放，占用了PG端的一条链接

TCP链接创建成功，却与PG通信超时，很有可能是PG负载过高的情况下，发生这种情况。

建议库没更新的情况下，不使用connect_timeout咯
