---
date: 2013-5-17
layout: default
title: 初架服务器

---
##初架服务器

###目的
构建一个服务器,一端监听HTTP请求,连接成功后用tcp请求另外一台服务器.简而言之,http请求转tcp.  

在http接受成功的线程里，new一个socket先sendrecv线程