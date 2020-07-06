---  
layout: post  
title: "zabbix5.0部署+配置"  
categories: docker zabbix  
tags: docker zabbix  
---  

# docker安装zabbix 5.0  
首先将/var/lib/docker/软连接到/data分区，以防以后空间不够用  
## 1. 安装mysql服务器  





```  
docker run --name mysql-server -t \  
      -v /etc/localtime:/etc/localtime \  
      -v /data/docker/mysql:/var/lib/mysql \  
      -e MYSQL_DATABASE="zabbix" \  
      -e MYSQL_USER="zabbix" \  
      -e MYSQL_PASSWORD="xxxx" \  
      -e MYSQL_ROOT_PASSWORD="xxxx" \  
      --restart unless-stopped \  
      -d mysql:8.0 \  
      --character-set-server=utf8 \  
      --collation-server=utf8_bin \  
      --default-authentication-plugin=mysql_native_password  
```  
## 2. 安装java网关  
```  
docker run --name zabbix-java-gateway -t \  
      -v /etc/localtime:/etc/localtime \  
      -v /data/docker/zabbix_java/ext_lib:/usr/sbin/zabbix_java/ext_lib \  
      --restart unless-stopped \  
      -d zabbix/zabbix-java-gateway:latest  
```  
## 3. 安装zabbix-server  

### 3.1 编译dockerfile  
我们这里编译dockerfile，生成centos-clatest的zabbix，因为要用python及requests包发送钉钉告警消息  

```  
FROM centos:centos7  

LABEL org.opencontainers.image.title="Zabbix server (MySQL)" \  
      org.opencontainers.image.authors="Alexey Pustovalov <alexey.pustovalov@zabbix.com>" \  
      org.opencontainers.image.vendor="Zabbix LLC" \  
      org.opencontainers.image.url="https://zabbix.com/" \  
      org.opencontainers.image.description="Zabbix server with MySQL database support" \  
      org.opencontainers.image.licenses="GPL v2.0"  

STOPSIGNAL SIGTERM  

ENV TINI_VERSION=v0.19.0  

RUN set -eux && \  
    groupadd -g 1995 --system zabbix && \  
    adduser -r --shell /sbin/nologin \  
            -g zabbix -G dialout -G root \  
            -d /var/lib/zabbix/ -u 1997 \  
        zabbix && \  
    mkdir -p /etc/zabbix && \  
    mkdir -p /var/lib/zabbix && \  
    mkdir -p /var/lib/zabbix/enc && \  
    mkdir -p /var/lib/zabbix/export && \  
    mkdir -p /var/lib/zabbix/mibs && \  
    mkdir -p /var/lib/zabbix/modules && \  
    mkdir -p /var/lib/zabbix/snmptraps && \  
    mkdir -p /var/lib/zabbix/ssh_keys && \  
    mkdir -p /var/lib/zabbix/ssl && \  
    mkdir -p /var/lib/zabbix/ssl/certs && \  
    mkdir -p /var/lib/zabbix/ssl/keys && \  
    mkdir -p /var/lib/zabbix/ssl/ssl_ca && \  
    mkdir -p /usr/lib/zabbix/alertscripts && \  
    mkdir -p /usr/lib/zabbix/externalscripts && \  
    mkdir -p /usr/share/doc/zabbix-server-mysql && \  
    yum --quiet makecache && \  
    yum -y install --setopt=tsflags=nodocs https://repo.zabbix.com/non-supported/rhel/7/x86_64/fping-3.10-1.el7.x86_64.rpm && \  
    yum -y install --setopt=tsflags=nodocs \  
            iputils \  
            traceroute \  
            libcurl \  
            libevent \  
            libxml2 \  
            mariadb \  
            net-snmp-libs \  
            OpenIPMI-libs \  
            openldap \  
            openssl-libs \  
            pcre \  
            unixODBC \  
            epel-release \  
            python3 && \  
    unlink /usr/bin/python && \  
    ln -s /usr/bin/python3 /usr/bin/python && \  
    sed -i '1s/python/python2/' /usr/bin/yum && sed -i '1s/python/python2/' /usr/libexec/urlgrabber-ext-down && \  
    ln -s /usr/bin/pip3 /usr/bin/pip && \  
    pip install requests && \  
    curl -L https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini -o /sbin/tini && \  
    curl -L https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini.asc -o /tini.asc && \  
    gpg --batch --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 595E85A6B1B4779EA4DAAEC70B588DFF0527A9B7 && \  
    gpg --batch --verify /tini.asc /sbin/tini && \  
    rm -rf /root/.gnupg && \  
    chmod +x /sbin/tini && \  
    yum -y clean all && \  
    rm -rf /var/cache/yum /var/lib/yum/yumdb/* /usr/lib/udev/hwdb.d/* && \  
    rm -rf /etc/udev/hwdb.bin /root/.pki  

ARG MAJOR_VERSION=5.0  
ARG ZBX_VERSION=${MAJOR_VERSION}.1  
ARG ZBX_SOURCES=https://git.zabbix.com/scm/zbx/zabbix.git  

ENV TERM=xterm MIBDIRS=/usr/share/snmp/mibs:/var/lib/zabbix/mibs MIBS=+ALL \  
    ZBX_VERSION=${ZBX_VERSION} ZBX_SOURCES=${ZBX_SOURCES}  

LABEL org.opencontainers.image.documentation="https://www.zabbix.com/documentation/${MAJOR_VERSION}/manual/installation/containers" \  
      org.opencontainers.image.version="${ZBX_VERSION}" \  
      org.opencontainers.image.source="${ZBX_SOURCES}"  

RUN set -eux && \  
    yum --quiet makecache && \  
    yum -y install --setopt=tsflags=nodocs \  
            autoconf \  
            automake \  
            gcc \  
            libcurl-devel \  
            libevent-devel \  
            libssh2-devel \  
            libxml2-devel \  
            make \  
            mariadb-devel \  
            net-snmp-devel \  
            OpenIPMI-devel \  
            openldap-devel \  
            git \  
            unixODBC-devel && \  
    cd /tmp/ && \  
    git clone ${ZBX_SOURCES} --branch ${ZBX_VERSION} --depth 1 --single-branch zabbix-${ZBX_VERSION} && \  
    cd /tmp/zabbix-${ZBX_VERSION} && \  
    zabbix_revision=`git rev-parse --short HEAD` && \  
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" include/version.h && \  
    ./bootstrap.sh && \  
    export CFLAGS="-fPIC -pie -Wl,-z,relro -Wl,-z,now" && \  
    ./configure \  
            --datadir=/usr/lib \  
            --libdir=/usr/lib/zabbix \  
            --prefix=/usr \  
            --sysconfdir=/etc/zabbix \  
            --enable-agent \  
            --enable-server \  
            --with-mysql \  
            --with-ldap \  
            --with-libcurl \  
            --with-libxml2 \  
            --with-net-snmp \  
            --with-openipmi \  
            --with-openssl \  
            --with-ssh2 \  
            --with-unixodbc \  
            --enable-ipv6 \  
            --silent && \  
    make -j"$(nproc)" -s dbschema && \  
    make -j"$(nproc)" -s && \  
    cp src/zabbix_server/zabbix_server /usr/sbin/zabbix_server && \  
    cp src/zabbix_get/zabbix_get /usr/bin/zabbix_get && \  
    cp src/zabbix_sender/zabbix_sender /usr/bin/zabbix_sender && \  
    cp conf/zabbix_server.conf /etc/zabbix/zabbix_server.conf && \  
    cat database/mysql/schema.sql > database/mysql/create.sql && \  
    cat database/mysql/images.sql >> database/mysql/create.sql && \  
    cat database/mysql/data.sql >> database/mysql/create.sql && \  
    gzip database/mysql/create.sql && \  
    cp database/mysql/create.sql.gz /usr/share/doc/zabbix-server-mysql/ && \  
    cd /tmp/ && \  
    rm -rf /tmp/zabbix-${ZBX_VERSION}/ && \  
    chown --quiet -R zabbix:root /etc/zabbix/ /var/lib/zabbix/ && \  
    chgrp -R 0 /etc/zabbix/ /var/lib/zabbix/ && \  
    chmod -R g=u /etc/zabbix/ /var/lib/zabbix/ && \  
    yum -y history undo `yum -q history | sed -n 3p |column -t | cut -d' ' -f1` && \  
    yum -y clean all && \  
    rm -rf /var/cache/yum /var/lib/yum/yumdb/* /usr/lib/udev/hwdb.d/* && \  
    rm -rf /etc/udev/hwdb.bin /root/.pki  

EXPOSE 10051/TCP  

WORKDIR /var/lib/zabbix  

VOLUME ["/usr/lib/zabbix/alertscripts", "/usr/lib/zabbix/externalscripts", "/var/lib/zabbix/enc", "/var/lib/zabbix/mibs", "/var/lib/zabbix/modules"]  
VOLUME ["/var/lib/zabbix/snmptraps", "/var/lib/zabbix/ssh_keys", "/var/lib/zabbix/ssl/certs", "/var/lib/zabbix/ssl/keys", "/var/lib/zabbix/ssl/ssl_ca"]  
VOLUME ["/var/lib/zabbix/export"]  
      
COPY ["docker-entrypoint.sh", "/usr/bin/"]  

ENTRYPOINT ["/sbin/tini", "--", "/usr/bin/docker-entrypoint.sh"]  

USER 1997  

CMD ["/usr/sbin/zabbix_server", "--foreground", "-c", "/etc/zabbix/zabbix_server.conf"]  

```  
打包会遇到p80超时问题，多尝试几次就好了  

### 3.2 关联实例,启动zabbix-server  
```  
私有仓库需要修改下docker配置，然后重启  
vi /etc/docker/daemon.json  
{"insecure-registries":["10.7.201.41:5000"]}  
systemctl restart docker.service  

docker run --name zabbix-server-mysql -t \  
      -v /etc/localtime:/etc/localtime \  
      -v /data/docker/zabbix/alertscripts:/usr/lib/zabbix/alertscripts \  
      -e DB_SERVER_HOST="mysql-server" \  
      -e MYSQL_DATABASE="zabbix" \  
      -e MYSQL_USER="zabbix" \  
      -e MYSQL_PASSWORD="xxxx" \  
      -e MYSQL_ROOT_PASSWORD="xxxx" \  
      -e ZBX_JAVAGATEWAY="zabbix-java-gateway" \  
      --link mysql-server:mysql \  
      --link zabbix-java-gateway:zabbix-java-gateway \  
      -p 10051:10051 \  
      --restart=unless-stopped \  
      -d 10.7.201.41:5000/zabbix-server-mysql:v1  
```  
## 4 启动web服务  

docker run --name zabbix-web-nginx-mysql -t \  
      -v /etc/localtime:/etc/localtime \  
      -e DB_SERVER_HOST="mysql-server" \  
      -e MYSQL_DATABASE="zabbix" \  
      -e MYSQL_USER="zabbix" \  
      -e MYSQL_PASSWORD="xxxx" \  
      -e MYSQL_ROOT_PASSWORD="xxxx" \  
      -e PHP_TZ="Asia/Shanghai" \  
      --link mysql-server:mysql \  
      --link zabbix-server-mysql:zabbix-server \  
      -p 80:8080 \  
      --restart=unless-stopped \  
      -d zabbix/zabbix-web-nginx-mysql:latest  

## 5 安装zabbix agent  
docker run --name zabbix-agent \  
            -e ZBX_HOSTNAME="10.7.201.41" \  
            -e ZBX_SERVER_HOST="10.7.201.98" \  
            -p 10050:10050 \  
            --privileged \  
            -d zabbix/zabbix-agent:latest  


# 监控配置  
## 网络设备监控  
### 1.网络设备开启snmp  
### 2.创建主机群组  
    配置-主机群组  
### 3.添加主机  
![20200702222111](https://guott.oss-cn-beijing.aliyuncs.com/img/20200702222111.png)  
### 4.配置主机宏  
![20200702222229](https://guott.oss-cn-beijing.aliyuncs.com/img/20200702222229.png)  
### 5.配置主机模板  
![20200702223401](https://guott.oss-cn-beijing.aliyuncs.com/img/20200702223401.png)  

## 主机设备监控  
当我们主机设备比较多时，可以配置自动发现来自动发现主机，并添加到主机组，应用模板、触发器等  
### 1.配置自动发现  
配置-自动发现-创建发现规则  
![20200702224633](https://guott.oss-cn-beijing.aliyuncs.com/img/20200702224633.png)  
### 2.配置自动发现动作  
配置-动作-自动发现触发器-创建动作  
这个是条件，过滤器的作用  
![20200703085353](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703085353.png)  
### 3.配置自动发现操作  
添加到主机群组，并链接到linux主机模板，这样从自动发现、采集、告警全自动化，非常方便  
![20200703085522](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703085522.png)  

## 报警推送  
### 1.创建报警媒介  
管理-报警媒介类型-创建媒介类型  
* 邮件告警  
1)创建邮件告警  
![20200703104808](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703104808.png)  
2)配置邮件告警模板  
![20200703103325](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703103325.png)  
> 问题模板  
##Alarm  
HOST: {HOST.NAME}  
IP: {HOST.IP}  
Alarm time: {EVENT.DATE} {EVENT.TIME}  
Problem name: {EVENT.NAME}  
Severity: {TRIGGER.SEVERITY}  
Status: {TRIGGER.STATUS}:{ITEM.VALUE1}  
Problem duration: {EVENT.AGE}  

> 恢复模板  
##Recover  
HOST: {HOST.NAME}  
IP: {HOST.IP}  
Recover time: {EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}  
Problem name: {EVENT.NAME}  
Severity: {TRIGGER.SEVERITY}  
Status: {TRIGGER.STATUS}:{ITEM.VALUE1}  
Problem duration: {EVENT.AGE}  

* 钉钉告警  
1)创建钉钉告警  
![20200703103611](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703103611.png)  
2)配置钉钉告警模板  
![20200703104518](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703104518.png)  
和邮箱模板保持一致即可  
3)钉钉自定义机器人  
群设置-智能群助手-添加机器人  
![20200703110940](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703110940.png)  

4)配置脚本  
将dingding.py脚本放置于zabbix服务端的/data/docker/zabbix/alertscripts目录下，实际是docker内部的/usr/lib/zabbix/alertscripts目录  
cat zabbix_server.conf  
AlertScriptsPath=/usr/lib/zabbix/alertscripts  
 
```  
#!/usr/bin/python  
#coding:utf-8  
import sys  
import time  
import hmac  
import hashlib  
import base64  
import urllib  
import json  
import requests  
import logging  

try:  
    JSONDecodeError = json.decoder.JSONDecodeError  
except AttributeError:  
    JSONDecodeError = ValueError  


def is_not_null_and_blank_str(content):  
    if content and content.strip():  
        return True  
    else:  
        return False  


class DingtalkRobot(object):  
    def __init__(self, webhook, sign=None):  
        super(DingtalkRobot, self).__init__()  
        self.webhook = webhook  
        self.sign = sign  
        self.headers = {'Content-Type': 'application/json; charset=utf-8'}  
        self.times = 0  
        self.start_time = time.time()  

    # 加密签名  
    def __spliceUrl(self):  
        timestamp = int(round(time.time() * 1000))  
        secret = self.sign  
        secret_enc = secret.encode('utf-8')  
        string_to_sign = '{}\n{}'.format(timestamp, secret)  
        string_to_sign_enc = string_to_sign.encode('utf-8')  
        hmac_code = hmac.new(secret_enc, string_to_sign_enc, digestmod=hashlib.sha256).digest()  
        sign = urllib.parse.quote_plus(base64.b64encode(hmac_code))  
        url = f"{self.webhook}&timestamp={timestamp}&sign={sign}"  
        return url  

    def send_text(self, msg, is_at_all=False, at_mobiles=[]):  
        data = {"msgtype": "text", "at": {}}  
        if is_not_null_and_blank_str(msg):  
            data["text"] = {"content": msg}  
        else:  
            logging.error("text类型，消息内容不能为空！")  
            raise ValueError("text类型，消息内容不能为空！")  

        if is_at_all:  
            data["at"]["isAtAll"] = is_at_all  

        if at_mobiles:  
            at_mobiles = list(map(str, at_mobiles))  
            data["at"]["atMobiles"] = at_mobiles  

        logging.debug('text类型：%s' % data)  
        return self.__post(data)  

    def __post(self, data):  
        """  
        发送消息（内容UTF-8编码）  
        :param data: 消息数据（字典）  
        :return: 返回发送结果  
        """  
        self.times += 1  
        if self.times > 20:  
            if time.time() - self.start_time < 60:  
                logging.debug('钉钉官方限制每个机器人每分钟最多发送20条，当前消息发送频率已达到限制条件，休眠一分钟')  
                time.sleep(60)  
            self.start_time = time.time()  

        post_data = json.dumps(data)  
        try:  
            response = requests.post(self.__spliceUrl(), headers=self.headers, data=post_data)  
        except requests.exceptions.HTTPError as exc:  
            logging.error("消息发送失败， HTTP error: %d, reason: %s" % (exc.response.status_code, exc.response.reason))  
            raise  
        except requests.exceptions.ConnectionError:  
            logging.error("消息发送失败，HTTP connection error!")  
            raise  
        except requests.exceptions.Timeout:  
            logging.error("消息发送失败，Timeout error!")  
            raise  
        except requests.exceptions.RequestException:  
            logging.error("消息发送失败, Request Exception!")  
            raise  
        else:  
            try:  
                result = response.json()  
            except JSONDecodeError:  
                logging.error("服务器响应异常，状态码：%s，响应内容：%s" % (response.status_code, response.text))  
                return {'errcode': 500, 'errmsg': '服务器响应异常'}  
            else:  
                logging.debug('发送结果：%s' % result)  
                if result['errcode']:  
                    error_data = {"msgtype": "text", "text": {"content": "钉钉机器人消息发送失败，原因：%s" % result['errmsg']},  
                                  "at": {"isAtAll": True}}  
                    logging.error("消息发送失败，自动通知：%s" % error_data)  
                    requests.post(self.webhook, headers=self.headers, data=json.dumps(error_data))  
                return result  


if __name__ == '__main__':  
    URL = "https://oapi.dingtalk.com/robot/send?access_token=44ccbc2eaccacebb96c2fe4ab5147a4412b92f4d9dab2fb37e48dee0a9b42d2c"  
    SIGN = "SEC0d4196a942928517210368b1870022b8705a004c3e7a26aa952d518ae0f8876b"  
    ding = DingtalkRobot(URL, SIGN)  
    message = sys.argv[1]  
    ding.send_text(message)  
```  
### 2.用户群组配置  
* 1）创建用户群组  
管理-用户群组-创建用户群组  
![20200703095504](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703095504.png)  
* 2）分配用户群组权限  
只准许主机组级别的用户组访问 Zabbix 中的任何主机数据。  
这意味着个人用户不能被直接授予对主机（或主机组）的访问权限。 个人用户只能通过其归属的用户组被授予主机组访问权限，进而访问该主机组下的主机 (即个人用户——>用户组——>主机组—–>主机)。  
但是实测用户设为超级管理员的话会拥有所有主机的读写权限  
![20200703095723](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703095723.png)  
### 3.用户配置  
* 1）创建用户  
管理-用户-创建用户  
![20200703095943](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703095943.png)  
* 2）配置用户报警媒介  
动作里面虽然指定的媒介，这里也需要配置上才行  
![20200703163111](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703163111.png)  
* 3）配置用户权限  
![20200703163149](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703163149.png)  

### 4. 报警推送  
* 1）创建动作触发条件  
配置-动作-选择触发器-创建动作  
警示度大于等于一般告警作为触发条件  
![20200703102645](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703102645.png)  
* 2）配置报警推送  
配置-动作-选择触发器-创建动作-操作  
选择相应的用户及媒介  
![20200703102704](https://guott.oss-cn-beijing.aliyuncs.com/img/20200703102704.png)  


# 问题  
## 1.Externalscripts need python - how to add this? #497  
https://github.com/zabbix/zabbix-docker/issues/497  


## 2.时区问题  
http://www.ishenping.com/artinfo/3393150.html  
