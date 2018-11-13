---
layout: post
title: "linux下rsync+sersync同步"
categories: sersync
tags: linux rsync sersync
---
* content
{:toc}



# 一、基本介绍 
## 1.Rsync

  Rsync（Remote Synchronize）是一款开源的、快速的、多功能的、可以实现全量及增量的本地或远程数据同步备份的优秀工具，并且支持多种操作系统平台运行。  
官网文档：https://rsync.samba.org/ftp/rsync/rsync.html

Rsync具有本地与远程两台主机之间的数据快速复制同步镜像、远程备份等功能，该功能类似scp，但是优于scp功能，还具有本地不同分区目录之间全量及增量复制数据。

Rsync同步数据镜像时，通过“quick check”算法，仅同步大小或最后修改时间发生变化的文件或目录，当然也可以根据权限，属主等属性变化的同步，所以可以实现快速同步。

rsync 具有如下的基本特性：

* 可以镜像保存整个目录树和文件系统

* 可以很容易做到保持原来文件的权限、时间、软硬链接等

* 无须特殊权限即可安装

* 优化的流程，文件传输效率高

* 可以使用 rsh、ssh 方式来传输文件，当然也可以通过直接的 socket 连接

* 支持匿名传输，以方便进行网站镜象

## 2.Sersync

sersync是基于inotify开发的，类似于inotify-tools的工具，Sersync可以记录下被监听目录中发生变化的（包括增加、删除、修改）具体某一个文件或者某一个目录的名字，然后使用rsync同步的时候，只同步发生变化的文件或者目录，因此效率更高。

主要应用场景为数据体积大，并且文件很多。


# 二、搭建步骤
## 系统环境
> 192.168.56.46 rsync 接收端 rsync

> 192.168.56.94 rsync 发送端rsync + sersync

> 将192.168.56.94的/data/nginx/resty/nginx/cert/证书目录变化实时推送给192.168.56.46

## 接收端配置
### 1.安装rsync
```
yum install rsync
```

### 2.编辑rsync配置文件
> /etc/rsyncd.conf 

```
uid = root
gid = root
read only =no
address = 192.168.56.46
log format= %h %o %f %l %b
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid

[nginx-cert]
        path = /data/nginx/resty/nginx/cert/
        list = yes
        hosts allow = 192.168.56.94
        hosts deny = *
        auth user = yuser
        secrets file = /etc/rsync.pas
```  
> /etc/rsync.pas

```
ypasswd 
```    
        
### 3.启动rsync 服务
```
rsync --daemon -4
```
* --daemon 守护启动  
* -4 ipv4

## 发送端配置
### 1.安装rsync
```
yum install rsync
```

### 2.安装sersync
```
wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/sersync/sersync2.5.4_64bit_binary_stable_final.tar.gz
tar -zxvf sersync2.5.4_64bit_binary_stable_final.tar.gz
mkdir -p /usr/local/sersync
mv  GNU-Linux-x86/{sersync2,confxml.xml} /usr/local/sersync/
```
### 3.编辑rsync配置文件
> /etc/rsync.pas

```
yuser:ypasswd
```

### 4.手动传输测试
 ```
 rsync -avzP aa rsync@192.168.56.46::nginx-cert --password-file=/etc/rsync.pas
 ```

### 5.编辑serync配置文件

> /usr/local/sersync/nginx-cert.xml  
[配置参数详解](https://cloud.tencent.com/info/498bf62a275ecba02751c184a3affabd.html) 

```
<head version="2.5">                                                                                    
    <host hostip="localhost" port="8008"></host>                                                        
    <debug start="false"/>                                                                              
    <fileSystem xfs="false"/>                                                                           
    <filter start="true">                                                                               
        <exclude expression="(.*)\.svn"></exclude>                                                      
        <exclude expression="(.*)\.pid"></exclude>                                                      
        <exclude expression="(.*)\.log"></exclude>
        <exclude expression="(.*)\.conf"></exclude>                                                      
        <exclude expression="(.*)\.env"></exclude>                                                      
        <exclude expression="(.*)\.header"></exclude>                                                                                                           
        <exclude expression="^/nginx/logs/*"></exclude>                                                 
        <exclude expression="^/ca"></exclude>
        <exclude expression="^/deploy"></exclude>
        <exclude expression="^/dnsapi"></exclude>
        <exclude expression="^static/*"></exclude>                                                                                                            
    </filter>                                                                                           
    <inotify>                                                                                           
        <delete start="true"/>                                                                          
        <createFolder start="true"/>                                                                    
        <createFile start="true"/>                                                                      
        <closeWrite start="true"/>                                                                      
        <moveFrom start="true"/>                                                                        
        <moveTo start="true"/>                                                                          
        <attrib start="true"/>                                                                          
        <modify start="true"/>                                                                         
    </inotify>                                                                                          
                                                                                                        
    <sersync>                                                                                           
        <localpath watch="/data/opers/.acme.sh/">                                                           
            <remote ip="192.168.56.46" name="nginx-cert"/>
        </localpath>                                                                                    
        <rsync>                                                                                         
            <commonParams params="-artuz"/>                                                             
            <auth start="false" users="root" passwordfile="/etc/rsync.pas"/>                            
            <userDefinedPort start="false" port="874"/><!-- port=874 -->                                
            <timeout start="false" time="100"/><!-- timeout=100 -->                                     
            <ssh start="false"/>                                                                        
        </rsync>                                                                                        
        <failLog path="/usr/local/sersync/sersync_fail.sh" timeToExecute="60"/><!--default every 60mins execute once-->
        <crontab start="false" schedule="600"><!--600mins-->                                            
            <crontabfilter start="false">                                                               
                <exclude expression="*.php"></exclude>                                                  
                <exclude expression="info/*"></exclude>                                                 
            </crontabfilter>                                                                            
        </crontab>                                                                                      
        <plugin start="false" name="command"/>                                                          
    </sersync>                                                                                          
                                                                                                        
    <plugin name="command">                                                                             
        <param prefix="/bin/sh" suffix="" ignoreError="true"/>  <!--prefix /opt/tongbu/mmm.sh suffix--> 
        <filter start="false">                                                                          
            <include expression="(.*)\.php"/>                                                           
            <include expression="(.*)\.sh"/>                                                            
        </filter>                                                                                       
    </plugin>                                                                                           
                                                                                                        
    <plugin name="socket">                                                                              
        <localpath watch="/opt/tongbu">                                                                 
            <deshost ip="192.168.138.20" port="8009"/>                                                  
        </localpath>                                                                                    
    </plugin>                                                                                           
    <plugin name="refreshCDN">                                                                          
        <localpath watch="/data0/htdocs/cms.xoyo.com/site/">                                            
            <cdninfo domainname="ccms.chinacache.com" port="80" username="xxxx" passwd="xxxx"/>         
            <sendurl base="http://pic.xoyo.com/cms"/>                                                   
            <regexurl regex="false" match="cms.xoyo.com/site([/a-zA-Z0-9]*).xoyo.com/images"/>          
        </localpath>                                                                                    
    </plugin>                                                                                           
</head>
```
### 6.启动sersync服务
/usr/local/sersync/sersync2 -d -r -o /usr/local/sersync/nginx-cert.xml                                                                                            






