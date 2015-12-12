---
layout: post
title: "letsencrypt实践"
categories:
  - web
---


随着互联网的发展，网络安全成了一个越来越重要的话题，而作为一个web开发者，使用https部署自己的站点成为了安全的重要一步，废话完毕。

不过话说回来，许多http2的实现都是只基于tls的。个人站点如果要配置https，就得去买https的证书，买是要化钱的，虽然也花不了几个钱。不过现在有免费的选择，那就是[Let’s Encrypt](https://letsencrypt.org/)。

先简单介绍一下Let’s Encrypt，他也是一家证书颁发机构（CA)，旨在通过提供免费、自动、开放的证书服务，以达到天下无贼的目的。工作过程也很简单，你有一个域名，想进行加密，请求Let’s Encrypt给你发一个证书，Let’s Encrypt首先需要确保这个域名是你的，确认之后会发放对应域名的整数，你就可以用这些整数配置你的web服务使用https了。

Let’s Encrypt验证域名的方式有两种，一是提供域名的DNS记录，二是提供一个域名下的资源。说一下第二种方式，当Let’s Encrypt的agent向Let’s Encrypt请求域名（example.com）的证书时，Let’s Encrypt会生成一个私钥和一个随机数，agent会对随机数进行加密，把加密之后的内容放到指定的路径（/.well-known/acme-challenge/）下，如果这个域名是你的，那么加密后的内容可以通过http://example.com/.well-known/acme-challenge/xxxx被访问到，Let’s Encrypt通过验证加密后的内容是否匹配就可以确定域名的拥有者了。

请求证书的方式也是有agent发起，会使用之前颁发的私钥做加密。真详细的描述可以看[这里](https://letsencrypt.org/howitworks/technology/)。

安装方式也很简单：


	git clone https://github.com/letsencrypt/letsencrypt
	cd letsencrypt
	./letsencrypt-auto

Let’s Encrypt提供了多种方式来获取和使用整数：apache、nginx、webroot、standlone。apache和nginx都是利用相应的模块进行整数的获取。standlone是指启动一个独立的web服务来获取整数，这需要你的机器上的80端口或443端口是可用的。webroot是只利用现有的web服务来获取整数。我使用了webroot这种方式，执行以下命令：
`letsencrypt certonly --webroot -w /home/docker/usr/share/nginx/html -d blog.bug1874.com`

生成的证书在/etc/letsencrypt/live/bug1874.com下，会生成一下几个文件cert.pem、chain.pem、fullchain.pem、privkey.pem。然后是配置nginx：

	listen 443 ssl;
	ssl_certificate /etc/letsencrypt/live/blog.bug1874.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/blog.bug1874.com/privkey.pem;

Apache的https配置我就不帖了，有需要的自己google去。

上面写的内容Let’s Encrypt的[文档](http://letsencrypt.readthedocs.org/en/latest/)上都有，比我说的清楚明白，感兴趣的同学可以去瞄瞄。
