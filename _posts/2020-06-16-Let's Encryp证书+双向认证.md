
---  
layout: post  
title: "ssl证书申请+双向认证"  
categories: nginx ssl  
tags: nginx ssl
---  
* content
{:toc}


# ssl证书申请并自动续期  
## 环境  
zys024 172.17.73.123  

## 安装acme.sh  
安装很简单, 一个命令:  
curl  https://get.acme.sh | sh




普通用户和 root 用户都可以安装使用. 安装过程进行了以下几步:  
* 1).把 acme.sh 安装到你的 home 目录下:  
~/.acme.sh/  
并创建 一个 bash 的 alias, 方便你的使用: alias acme.sh=~/.acme.sh/acme.sh  
* 2). 自动为你创建 cronjob, 每天 0:00 点自动检测所有的证书, 如果快过期了, 需要更新, 则会自动更新证书.  
更高级的安装选项请参考: https://github.com/Neilpang/acme.sh/wiki/How-to-install  
安装过程不会污染已有的系统任何功能和文件, 所有的修改都限制在安装目录中: ~/.acme.sh/  

## 创建阿里云api key,授权dns解析管理权限  
通过获取token，api自动添加和删除dns记录  
配置环境变量key  
export Ali_Key="xxxxx"  
export Ali_Secret="xxx"  
 这里给出的 api id 和 api key 会被自动记录下来, 将来你在使用 dnspod api 的时候, 就不需要再次指定了. 直接生成就好了  

## 申请证书  
/root/.acme.sh/acme.sh --issue --dns dns_ali -d psbsafenter.cn -d *.psbsafenter.cn  
-----END CERTIFICATE-----  
[Tue Jun 11 14:55:50 CST 2019] Your cert is in  /root/.acme.sh/psbsafenter.cn/psbsafenter.cn.cer  
[Tue Jun 11 14:55:50 CST 2019] Your cert key is in  /root/.acme.sh/psbsafenter.cn/psbsafenter.cn.key  
[Tue Jun 11 14:55:50 CST 2019] v2 chain.  
[Tue Jun 11 14:55:50 CST 2019] The intermediate CA cert is in  /root/.acme.sh/psbsafenter.cn/ca.cer  
[Tue Jun 11 14:55:50 CST 2019] And the full chain certs is there:  /root/.acme.sh/psbsafenter.cn/fullchain.cer  
[Tue Jun 11 14:55:50 CST 2019] _on_issue_success  

## copy证书  
```  
mkdir -p /usr/local/nginx/cert/psbsafenter.cn  
./acme.sh  --installcert  -d  psbsafenter.cn -d *.psbsafenter.cn \  
        --key-file   /usr/local/nginx/cert/psbsafenter.cn/psbsafenter.cn.key \  
        --fullchain-file /usr/local/nginx/cert/psbsafenter.cn/fullchain.cer \  
        --reloadcmd  "/usr/local/nginx/sbin/nginx -s reload"  
```  
## 证书自动续期  
crontab -l  
42 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null  

默认添加了自动更新任务，无需人工去干预。证书有效期为90天，提前30天会去自动生成证书，--installcert命令也会跟着生效，使nginx重新加载最新证书。  

## nginx配置  
cat /usr/local/nginx/conf/vhosts/newsecurity.conf  

```  
server {  
    listen      80;  
    listen      443 ssl;  
    server_name new.psbsafenter.cn;  

    include /usr/local/nginx/conf/ssl/ssl_psbsafenter.cn.conf;  

    error_page   500 502 503 504  /50x.html;  
    location = /50x.html {  
        root   html;  
    }  

    location / {  
        #if ( $scheme = http ) {  
        #  rewrite  ^(.*)$ https://$host$1 permanent;  
        #}  
        proxy_pass http://newsecurity.zy.com:8080;  
        proxy_set_header   Host $host:$server_port;  
        proxy_set_header  X-Real-IP $remote_addr;  
        proxy_set_header  X-Real-Remote-Port $remote_port;  
        proxy_set_header  X-Real-Server-Port $server_port;  
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;  
        proxy_set_header  X-Forwarded-Proto $scheme;  
        proxy_http_version 1.1;  
        proxy_set_header Connection "";  
        proxy_redirect http://$host $scheme://$http_host;  
    }  
}  
```  
cat /usr/local/nginx/conf/ssl/ssl_psbsafenter.cn.conf  
```  
ssl_certificate   /usr/local/nginx/cert/psbsafenter.cn/fullchain.cer;  
ssl_certificate_key  /usr/local/nginx/cert/psbsafenter.cn/psbsafenter.cn.key;  
ssl_session_timeout 5m;  
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;  
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  
ssl_prefer_server_ciphers on;  
```  
# 双向证书认证  
## 根证书自签：  
#openssl genrsa -out ca-key.pem 2048  
#openssl req -new -key ca-key.pem -out ca-req.csr -days 3650  
```  
    You are about to be asked to enter information that will be incorporated  
    into your certificate request.  
    What you are about to enter is what is called a Distinguished Name or a DN.  
    There are quite a few fields but you can leave some blank  
    For some fields there will be a default value,  
    If you enter '.', the field will be left blank.  
    -----  
    Country Name (2 letter code) [XX]:cn  
    State or Province Name (full name) []:shandong     
    Locality Name (eg, city) [Default City]:jinan  
    Organization Name (eg, company) [Default Company Ltd]:spdb  
    Organizational Unit Name (eg, section) []:spdb  
    Common Name (eg, your name or your server's hostname) []:spdb  
    Email Address []:  
    
    Please enter the following 'extra' attributes  
    to be sent with your certificate request  
    A challenge password []:test  
    An optional company name []:test  
 #openssl x509 -req -in ca-req.csr -out ca-cert.pem -signkey ca-key.pem -days 3650  
 ```  
 
## 客户端证书：  
#openssl genrsa -out client-key.pem 2048  
#openssl req -new -key client-key.pem -out client-req.csr -days 3650  
```  
    You are about to be asked to enter information that will be incorporated  
    into your certificate request.  
    What you are about to enter is what is called a Distinguished Name or a DN.  
    There are quite a few fields but you can leave some blank  
    For some fields there will be a default value,  
    If you enter '.', the field will be left blank.  
    -----  
    Country Name (2 letter code) [XX]:cn  
    State or Province Name (full name) []:shandong     
    Locality Name (eg, city) [Default City]:jinan  
    Organization Name (eg, company) [Default Company Ltd]:spdb  
    Organizational Unit Name (eg, section) []:spdb  
    Common Name (eg, your name or your server's hostname) []:chengtou  
    Email Address []:  
    
    Please enter the following 'extra' attributes  
    to be sent with your certificate request  
    A challenge password []:test  
    An optional company name []:test  
```  
#openssl x509 -req -in client-req.csr -out client-cert.pem -days 3650 -CA ca-cert.pem  -CAkey ca-key.pem -CAcreateserial  
#openssl pkcs12 -export -clcerts -in client-cert.pem -inkey client-key.pem -out client.p12  
 
## 修改nginx配置文件  
#vi /usr/local/resty/nginx/conf/nginx.conf  
        listen      443;  
        server_name  tzbill.aiopos.cn;  

        ssl                  on;  
        ssl_certificate      /pfcb/nginx/resty/nginx/cert/tzbill.aiopos.cn.pem;  
        ssl_certificate_key  /pfcb/nginx/resty/nginx/cert/tzbill.aiopos.cn.key;  
        ssl_client_certificate /pfcb/nginx/resty/nginx/cert/private/ca-cert.pem;  

        ssl_session_timeout  5m;  
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;  
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  
        ssl_prefer_server_ciphers On; #指定服务器密码算法在优先于客户端密码算法时，使用SSLv3和TLS协议。  
        ssl_verify_client on;  
}  
## 重启nginx  
/usr/local/resty/nginx/sbin/nginx -s reload  
## 浏览器导入  
证书有问题会报400错误，调试不成功可以清理浏览器缓存、SSL缓存，重启浏览器尝试  
* 注意事项  
1、客户端的证书与根证书的Common Name不能相同，否则会报错如下：  
[alert] 10587#0: *3 ignoring stale global SSL error (SSL: error:0407006A:rsa routines:RSA_padding_check_PKCS1_type_1:block type is not 01 error:04067072:rsa routines:RSA_EAY_PUBLIC_DECRYPT:padding check failed error:0D0C5006:asn1 encoding routines:ASN1_item_verify:EVP lib) while waiting for request, client: 60.208.23.2, server: 0.0.0.0:443  

## 特定url的双向认证  
我们利用ssl_verify_client optional来实现特定URL的认证，前提是用户需要提前配置好客户端证书，对有需要的url通过if条件来判断，成功进入正常业务流程，失败直接返回错误  
###ssl双向测试###  
server {  
    listen       443 ssl;  
    server_name test.52wandoumiao.cn;  
    ssl_certificate   /usr/local/nginx/cert/52wandoumiao.cn/fullchain.cer;  
    ssl_certificate_key  /usr/local/nginx/cert/52wandoumiao.cn/52wandoumiao.cn.key;  
    ssl_client_certificate /usr/local/nginx/cert/client/ca-cert.pem;  
    ssl_session_timeout 5m;  
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;  
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;  
    ssl_prefer_server_ciphers on;  
    ssl_verify_client optional;  
    #ssl_verify_client on;  

    default_type text/html ;  
    location ^~ /ssl {  
        if ($ssl_client_verify != SUCCESS) {  
           return 403;  
           break;  
        }  
        return 200 /ssl;  
    }  

    location  / {  
        return 200 "/";  
    }  

    location  /1 {  
        return 200 "/1";  
    }  

    location  /2 {  
        return 200 "/2";  
    }  

