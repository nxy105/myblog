# 一个问题引发的 Nginx Rewrite 和 FastCGI 模块探索

## 问题背景

某日，线上出现了一个小问题。我们的 iOS 应用在上一次的发版中，由于测试不完整导致部分功能请求的接口 URL 出现了错误，业务功能受到了影响。鉴于 iOS 的审核机制，我们决定在服务端做兼容。

方案其实很简单，将类似如下请求转向新的地址：

```
GET /foo => GET /foo/bar
```

## 问题分析

很简单的，我们想到使用 Nginx 的 Rewrite 模块进行内部重定向。

```
server {
    listen 80;
    
    ...

	# 为了方便调试，我们打开 Nginx 的 Rewrite 模块日志
    rewrite_log on;
    error_log /var/log/nginx-error.log notice;

	# 当域名匹配 /foo 时，定向到 /foo/bar
    location = /foo {
        rewrite ^(.*)$ /foo/bar;
    }
    
    ...

    location ~ \.php$ {
        fastcgi_pass  127.0.0.1:9000;
        include       fastcgi_params;
    }

    location / {
        rewrite ^(.*)$ /index.php;
    }
}
```

增加以上配置之后，我们满怀信心的打开 iOS 应用测试，发现问题并没有解决！这时，我们在配置中打开的日志派上了用场，马上来查看日志中的内容：

```
2016/07/28 23:16:11 [notice] 67935#0: *3 "^(.*)$" matches "/foo", client: 127.0.0.1, server: my.dev, request: "GET /foo HTTP/1.1", host: "my.dev"
2016/07/28 23:16:11 [notice] 67935#0: *3 rewritten data: "/foo/bar", args: "", client: 127.0.0.1, server: my.dev, request: "GET /foo HTTP/1.1", host: "my.dev"
2016/07/28 23:16:11 [notice] 67935#0: *3 "^(.*)$" matches "/foo/bar", client: 127.0.0.1, server: my.dev, request: "GET /foo HTTP/1.1", host: "my.dev"
2016/07/28 23:16:11 [notice] 67935#0: *3 rewritten data: "/index.php", args: "", client: 127.0.0.1, server: my.dev, request: "GET /foo HTTP/1.1", host: "my.dev"
```

我们发现，重定向的匹配如我们所预期的进行，那为什么程序依然不能被正确的执行呢？这时候，我注意到日志中的`... request: "GET /foo HTTP/1.1" ...`这部分信息，从匹配到结束并没有发生改变。而后端接收服务的 PHP 程序是这样做路由匹配的：

```
<?php
if ($router->match($_SERVER['REQUEST_URI'])) {
	// 处理正确的业务逻辑
}
```

在程序中将`$_SERVER['REQUEST_URI']`变量打印到日志中发现，变量的值并没有如预期般被重定向为`/foo/bar`，而是保持原始值`/foo`，这样显然不是我们希望的结果。

尝试在 Nginx 修改`$request_uri`的值：

```
server {
	... 
	
	location = /foo {
        set $request_uri "/foo/bar";
        rewrite ^(.*)$ /foo/bar;
    }
    
    ...
}
```

在启动 Nginx 时，收到这样的错误：

```
nginx: [emerg] the duplicate "request_uri" variable in /usr/local/etc/nginx/servers/test.conf:18
```

现在，让我们重新分析当下的问题，我们希望在 PHP 的程序中的到`$_SERVER['REQUEST_URI']`变量是我们重定向后的值。这个场景中，Nginx 通过 FastCGI 协议和 PHP-FPM 程序进行通信。而`$_SERVER['REQUEST_URI']`是通过在 Nginx 中配置的参数传递的：

```
fastcgi_param  REQUEST_URI        $request_uri;
```

可以看到，默认值就是 Nginx 的`$request_uri`变量。于是，我们通过在 Nginx 设置新的变量，再将变量传递给 PHP-FPM。Nginx 配置如下：

```
server {
	... 
	
	set $request_url $request_uri;
    location = /foo {
        set $request_url /foo/bar;
        rewrite ^(.*)$ /foo/bar;
    }
    
    location ~ \.php$ {
    	...
    	
        # 必须先 include fastcgi_params 后设置 REQUEST_URI 变量，否则会被覆盖
        include       fastcgi_params;
        fastcgi_param REQUEST_URI $request_url;
        
        ...
    }
}
```

## 结语

这只是非常简单的一个问题，但通过这个小问题，我们可以从中收获到：

- 适当的日志能够帮助我们快速定位问题。
- 时刻保持对当下问题的焦点，不在一个错误的方向执着。（分析为什么 Nginx 无法设置`$request_uri`就是一个与问题相关性不大的方向）。
