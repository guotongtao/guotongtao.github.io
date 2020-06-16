---  
layout: post  
title: "开启nginx的gzip无效分析"  
categories: nginx  
tags: nginx gzip  
---  

* content  
{:toc}  

## 起因  
测试反应页面打开慢，大概要5s左右的样子，开启Chrome的调试模式发现，几M的静态文件没有用gzip传输，本文主要分析没有走gzip的原因  




## 确认  
![20200616112628](https://guott.oss-cn-beijing.aliyuncs.com/img/20200616112628.png)  
可以发现js文件占用了一定时间，当然还有其他问题  

## 排查  
* 1.检查nginx配置文件  
```  
    gzip  on;  
    gzip_types application/font-woff application/x-font-ttf application/javascript application/x-javascript application/json applicati  
on/xml text/json text/plain text/css text/xml text/javascript;  
    gzip_proxied any;  
    gzip_comp_level 5;  
    gzip_min_length 1024 ;  
```  
显然没问题，针对网上有说不生效是gzip_types没有包含类型的原因，显然是不适合我的问题的  

* 2.浏览器调试查看  
![20200616112923](https://guott.oss-cn-beijing.aliyuncs.com/img/20200616112923.png)  
请求头中有Accept-Encoding: gzip，服务器返回应该按照gzip相应  

* 3.抓包看服务端  
tcpdump host 6tcpdump host  124.133.3.194 and port 90  -i ens192 -w /tmp/90.cap  
![20200616113202](https://guott.oss-cn-beijing.aliyuncs.com/img/20200616113202.png)  
问题发现了，服务器端接受的头里面没有Accept-Encoding: gzip，按照网上解释很可能被waf给拦截掉了，我们环境正好是政务云，并且之前出现过很多的waf造成问题，联系浪潮处理  