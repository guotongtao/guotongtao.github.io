---
layout: post
title: acme.sh申请Let's Encrypt免费ssl证书
categories: ssl证书
tags:  acme LetsEncrypt
---

* content
{:toc}




# 参考
github:[https://github.com/Neilpang/acme.sh](github:https://github.com/Neilpang/acme.sh)

# 说明
acme.sh 实现了 acme 协议, 可以从 letsencrypt 生成免费的证书.

# 主要步骤

> 安装 acme.sh  
> 生成证书  
> copy 证书到 nginx/apache 或者其他服务  
> 更新证书  
> 更新 acme.sh  
> 出错怎么办, 如何调试  

# 详细介绍
## 1. 安装 acme.sh
安装很简单, 一个命令:
curl  https://get.acme.sh | sh
普通用户和 root 用户都可以安装使用. 安装过程进行了以下几步:
把 acme.sh 安装到你的 home 目录下:
~/.acme.sh/
并创建 一个 bash 的 alias, 方便你的使用: alias acme.sh=~/.acme.sh/acme.sh

2). 自动为你创建 cronjob, 每天 0:00 点自动检测所有的证书, 如果快过期了, 需要更新, 则会自动更新证书.

更高级的安装选项请参考: [https://github.com/Neilpang/acme.sh/wiki/How-to-install](https://github.com/Neilpang/acme.sh/wiki/How-to-install)

安装过程不会污染已有的系统任何功能和文件, 所有的修改都限制在安装目录中: ~/.acme.sh/

## 2. 生成证书
acme.sh 实现了 acme 协议支持的所有验证协议. 一般有两种方式验证: http 和 dns 验证.

1). http 方式需要在你的网站根目录下放置一个文件, 来验证你的域名所有权,完成验证. 然后就可以生成证书了.
acme.sh  --issue  -d mydomain.com -d www.mydomain.com  --webroot  /home/wwwroot/mydomain.com/
只需要指定域名, 并指定域名所在的网站根目录. acme.sh 会全自动的生成验证文件, 并放到网站的根目录, 然后自动完成验证. 最后会聪明的删除验证文件. 整个过程没有任何副作用.

如果你用的 apache服务器, acme.sh 还可以智能的从 apache的配置中自动完成验证, 你不需要指定网站根目录:

acme.sh --issue  -d mydomain.com   --apache
如果你用的 nginx服务器, 或者反代, acme.sh 还可以智能的从 nginx的配置中自动完成验证, 你不需要指定网站根目录:

acme.sh --issue  -d mydomain.com   --nginx
注意, 无论是 apache 还是 nginx 模式, acme.sh在完成验证之后, 会恢复到之前的状态, 都不会私自更改你本身的配置. 好处是你不用担心配置被搞坏, 也有一个缺点, 你需要自己配置 ssl 的配置, 否则只能成功生成证书, 你的网站还是无法访问https. 但是为了安全, 你还是自己手动改配置吧.

如果你还没有运行任何 web 服务, 80 端口是空闲的, 那么 acme.sh 还能假装自己是一个webserver, 临时听在80 端口, 完成验证:

acme.sh  --issue -d mydomain.com   --standalone
更高级的用法请参考: [https://github.com/Neilpang/acme.sh/wiki/How-to-issue-a-cert](https://github.com/Neilpang/acme.sh/wiki/How-to-issue-a-cert)

2). dns 方式, 在域名上添加一条 txt 解析记录, 验证域名所有权.
这种方式的好处是, 你不需要任何服务器, 不需要任何公网 ip, 只需要 dns 的解析记录即可完成验证. 坏处是，如果不同时配置 Automatic DNS API，使用这种方式 acme.sh 将无法自动更新证书，每次都需要手动再次重新解析验证域名所有权。

acme.sh  --issue  --dns   -d mydomain.com
然后, acme.sh 会生成相应的解析记录显示出来, 你只需要在你的域名管理面板中添加这条 txt 记录即可.

等待解析完成之后, 重新生成证书:

acme.sh  --renew   -d mydomain.com
注意第二次这里用的是 --renew

dns 方式的真正强大之处在于可以使用域名解析商提供的 api 自动添加 txt 记录完成验证.

acme.sh 目前支持 cloudflare, dnspod, cloudxns, godaddy 以及 ovh 等数十种解析商的自动集成.

以 dnspod 为例, 你需要先登录到 dnspod 账号, 生成你的 api id 和 api key, 都是免费的. 然后:

export DP_Id="1234"

export DP_Key="sADDsdasdgdsf"

acme.sh   --issue   --dns dns_dp   -d aa.com  -d www.aa.com

证书就会自动生成了. 这里给出的 api id 和 api key 会被自动记录下来, 将来你在使用 dnspod api 的时候, 就不需要再次指定了. 直接生成就好了:

acme.sh  --issue   -d  mydomain2.com   --dns  dns_dp
更详细的 api 用法: [https://github.com/Neilpang/acme.sh/blob/master/dnsapi/README.md](https://github.com/Neilpang/acme.sh/blob/master/dnsapi/README.md)

## 3. copy/安装 证书
前面证书生成以后, 接下来需要把证书 copy 到真正需要用它的地方.

注意, 默认生成的证书都放在安装目录下: ~/.acme.sh/, 请不要直接使用此目录下的文件, 例如: 不要直接让 nginx/apache 的配置文件使用这下面的文件. 这里面的文件都是内部使用, 而且目录结构可能会变化.

正确的使用方法是使用 --installcert 命令,并指定目标位置, 然后证书文件会被copy到相应的位置, 例如:

acme.sh  --installcert  -d  <domain>.com   \
        --key-file   /etc/nginx/ssl/<domain>.key \
        --fullchain-file /etc/nginx/ssl/fullchain.cer \
        --reloadcmd  "service nginx force-reload"
(一个小提醒, 这里用的是 service nginx force-reload, 不是 service nginx reload, 据测试, reload 并不会重新加载证书, 所以用的 force-reload)

Nginx 的配置 ssl_certificate 使用 /etc/nginx/ssl/fullchain.cer ，而非 /etc/nginx/ssl/<domain>.cer ，否则 SSL Labs 的测试会报 Chain issues Incomplete 错误。

--installcert命令可以携带很多参数, 来指定目标文件. 并且可以指定 reloadcmd, 当证书更新以后, reloadcmd会被自动调用,让服务器生效.

详细参数请参考: [https://github.com/Neilpang/acme.sh#3-install-the-issued-cert-to-apachenginx-etc](https://github.com/Neilpang/acme.sh#3-install-the-issued-cert-to-apachenginx-etc)

值得注意的是, 这里指定的所有参数都会被自动记录下来, 并在将来证书自动更新以后, 被再次自动调用.

## 4. 更新证书
目前证书在 60 天以后会自动更新, 你无需任何操作. 今后有可能会缩短这个时间, 不过都是自动的, 你不用关心.

## 5. 更新 acme.sh
目前由于 acme 协议和 letsencrypt CA 都在频繁的更新, 因此 acme.sh 也经常更新以保持同步.

升级 acme.sh 到最新版 :

acme.sh --upgrade
如果你不想手动升级, 可以开启自动升级:

acme.sh  --upgrade  --auto-upgrade
之后, acme.sh 就会自动保持更新了.

你也可以随时关闭自动更新:

acme.sh --upgrade  --auto-upgrade  0
## 6. 出错怎么办：
如果出错, 请添加 debug log：

acme.sh  --issue  .....  --debug 
或者：

acme.sh  --issue  .....  --debug  2
请参考： [https://github.com/Neilpang/acme.sh/wiki/How-to-debug-acme.sh](https://github.com/Neilpang/acme.sh/wiki/How-to-debug-acme.sh)

最后, 本文并非完全的使用说明, 还有很多高级的功能, 更高级的用法请参看其他 wiki 页面.

[https://github.com/Neilpang/acme.sh/wiki](https://github.com/Neilpang/acme.sh/wiki)

Buy me a beer, Donate to acme.sh if it saves your time. Your donation makes acme.sh better: https://donate.acme.sh/

如果 acme.sh 帮你节省了时间,请考虑赏我一杯咖啡, 捐助: https://donate.acme.sh/ 你的支持将会使得 acme.sh 越来越好. 感谢



[https://github.com/Neilpang/acme.sh/wiki](https://github.com/Neilpang/acme.sh)

dnspod:
export DP_Id="69653"
export DP_Key="399853531f449f4e233f46793e72dea3"
acme.sh --issue --dns dns_dp -d catail.cn -d *.catail.cn --debug
acme.sh --issue --dns dns_dp -d aiopos.cn -d *.aiopos.cn --debug
acme.sh --issue --dns dns_dp -d aoepos.cn -d *.aoepos.cn  --debug
acme.sh --issue --dns dns_dp -d aoepos.com -d *.aoepos.com --debug


godaddy:
export GD_Key="dKYQLM3YuL8g_3tvhy7etGoKejZbUHrCVh1"
export GD_Secret="3tw37Q6CyiBHkrBYCtQgxo"

acme.sh --issue --dns dns_gd -d cmstech.sg -d *.cmstech.sg

# 实例：
以godaddy为例
## 获取token:
通过获取token，api自动添加和删除dns记录，获取token具体方法参考godaddy帮助手册
## 环境变量
export GD_Key="dKYQLM3YuL8gxxx"  
export GD_Secret="3tw37xxx"

环境变量只需要执行一次，执行成功后会在acccount.conf文件中记录下api key,多个dns解析商以不同的变量命名，如DNSPOD变量为SAVED_DP_Id，SAVED_DP_Key
> cat account.conf
SAVED_GD_Key='dKYQLM3YuL8g_3tvhy7etGoKejZbUHrCVh1'
SAVED_GD_Secret='3tw37Q6CyiBHkrBYCtQgxo'

## 生成证书
> acme.sh --issue --dns dns_dp -d example.sg -d *.example.sg --debug
```
[Tue Nov 20 10:57:22 CST 2018] Creating domain key  
[Tue Nov 20 10:57:22 CST 2018] The domain key is here: /data/opers/.acme.sh/example.sg/example.sg.key  
[Tue Nov 20 10:57:22 CST 2018] Multi domain='DNS:cmstech.sg,DNS:*.cmstech.sg'  
[Tue Nov 20 10:57:22 CST 2018] Getting domain auth token for each domain 
[Tue Nov 20 10:57:25 CST 2018] Getting webroot for domain='cmstech.sg'
[Tue Nov 20 10:57:25 CST 2018] Getting webroot for domain='*.cmstech.sg'
[Tue Nov 20 10:57:25 CST 2018] Found domain api file: /data/opers/.acme.sh/dnsapi/dns_gd.sh
[Tue Nov 20 10:57:28 CST 2018] Adding record
[Tue Nov 20 10:57:28 CST 2018] Added, sleeping 10 seconds
[Tue Nov 20 10:57:40 CST 2018] Found domain api file: /data/opers/.acme.sh/dnsapi/dns_gd.sh
[Tue Nov 20 10:57:41 CST 2018] Adding record
[Tue Nov 20 10:57:42 CST 2018] Added, sleeping 10 seconds
[Tue Nov 20 10:57:53 CST 2018] Sleep 120 seconds for the txt records to take effect
[Tue Nov 20 10:59:55 CST 2018] Verifying:cmstech.sg
[Tue Nov 20 10:59:58 CST 2018] Success
[Tue Nov 20 10:59:58 CST 2018] Verifying:*.cmstech.sg
[Tue Nov 20 11:00:01 CST 2018] Success
[Tue Nov 20 11:00:01 CST 2018] Removing DNS records.
[Tue Nov 20 11:00:06 CST 2018] Verify finished, start to sign.
[Tue Nov 20 11:00:13 CST 2018] Cert success.
```

生成的证书文件
> ls /data/opers/.acme.sh/example.sg  
```
ca.cer  example.sg.cer  example.sg.conf  example.sg.csr  example.sg.csr.conf  example.sg.key  fullchain.cer
```

如果再次执行会提示下次自动更新时间，除非用--force参数强制生成  
```
[Tue Nov 20 14:46:26 CST 2018] Renew: 'cmstech.sg'                                     
[Tue Nov 20 14:46:26 CST 2018] Skip, Next renewal time is: Sat Jan 19 03:00:13 UTC 2019
[Tue Nov 20 14:46:26 CST 2018] Add '--force' to force to renew.
```
 ## 定时任务
 安装脚本时默认已添加  
 > crontab -l 
 ```
 25 0 * * * "/data/opers/.acme.sh"/acme.sh --cron --home "/data/opers/.acme.sh" > /dev/null
 ```
 ## nginx配置
> vim s/data/nginx/resty/nginx/conf/nginx.conf  
   ```
   listen      443 ssl;        
   server_name example.sg;
   include /data/nginx/resty/nginx/conf/ssl/ssl_example.sg.conf;
   ```
  
> vim s/data/nginx/resty/nginx/conf/ssl/ssl_example.sg.conf 
  ```
  ssl_certificate   /data/nginx/resty/nginx/cert/example.sg/fullchain.cer;
  ssl_certificate_key  /data/nginx/resty/nginx/cert/example.sg/aiopos.cn.key;
  ssl_session_timeout 5m;
  ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
   ```

 ## 推送服务
 
 参考:[sersync+rsync实时同步](https://www.guotongtao.com/2018/10/19/rsynnc+sersync%E5%90%8C%E6%AD%A5/)
 
 **发送端**  
 sersync实时检测data/opers/.acme.sh目录下的证书更新，并通过rsync传输到接收端的nginx证书目录下
 /usr/local/sersync/sersync2 -d -r -o /usr/local/sersync/nginx-cert.xml
 
 **接收端**  
 /usr/bin/rsync --daemon -4
 
 ## 监控服务  
 证书生成脚本已经放在crontab中执行  
 监控sersync2和rsync保证生产业务能获取到最新的证书
