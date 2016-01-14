---
layout: post
title:  GET和POST的最大限制  
date:   2016-01-14
categories: Web
---

* GET  
HTTP协议对GET没有做明确的限制(超出可以返回414), 所以这个最大长度主要是Server, Client或者Proxy决定的.  
[RFC 2616](http://tools.ietf.org/html/rfc2616) (Hypertext Transfer Protocol — HTTP/1.1)
> there is no limit to the length of an URI  

	* Server  
	通常可以配置这个值, 常见的:
		* Apache: 源码 `DEFAULT_LIMIT_REQUEST_LINE` 的值来规定最大值, `LimitRequestLine` 可以在运行期决定最大值, 但不能超过前者
		* Nginx: `large_client_header_buffers` [参考](http://nginx.org/r/large_client_header_buffers)
	* Client  
	不同的Client限制不同, 2K以下通常是安全的, 常见的:
		* Firefox(1.5): >64K
		* Safari: >80K
		* Chrome: 2M
		* Edge: ~80K
		* IE4~:2K  
		
		上面数据来源不一定准确, 理论上 Chrome, Safari 都是基于WebKit, 限制应该是一样的
* POST
和GET一样, 协议没有做明确限制, limit=MIN(Server limit, Client limit)
	* Server
		* Nginx: `client_max_body_size`  
		* PHP: `post_max_size`  
		* Apache: `LimitRequestBody` [参考](http://httpd.apache.org/docs/2.0/mod/core.html#limitrequestbody)

		> This directive specifies the number of bytes from 0 (meaning unlimited) to 2147483647 (2GB) that are allowed in a request body.  <br/>

	* Client [参考](http://www.motobit.com/help/scptutl/pa98.htm)
		* Firefox: 2G
		* Safari: >4GB
		* IE3~: 2G

		可以看出来, POST的主要限制在Server, 一般Server不会接受这么大(G量级)的POST



