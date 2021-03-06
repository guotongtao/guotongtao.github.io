---  
layout: post  
title: "window下部署oracle12"  
categories: roacle  
tags: window oracle  
---  

* content  
{:toc}  

## *\*一、环境\*  

| ***\*名称\**** | ***\*参数\****                                       | ***\*备注\**** |  
| -------------- | ---------------------------------------------------- | -------------- |  
| 数据库名       | orcltest                                             |                |  
| 服务名         | orcltest                                             |                |  
| SID            | orcltest                                             |                |  
| 安装目录       | E:\app\Administrator\virtual\product\12.2.0\dbhome_1 | 系统默认       |  
| 数据文件目录   | E:\app\Administrator\virtual\oradata                 | 系统默认       |  





 

## ***\*二、安装步骤\****  

在文件夹中打开setup.exe,执行安装程序后会进入安装图形界面。  

1、不需要接收消息  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps1.jpg)  



2、创建和配置数据库  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps2.jpg)  

 

3、选择服务器类  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps3.jpg)  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps4.jpg)  

 

4、单实例数据库安装  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps5.jpg)  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps6.jpg)  

5、使用虚拟账号  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps7.jpg)  

 

 

 

6、基目录和软件位置，只修改根盘符  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps8.jpg)  

 

7、一般用途/事务处理  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps9.jpg)  

 

8、设置数据库的名称orcltest，取消创建为容器数据库  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps10.jpg)  

 

9、设置内存，根据服务器规划  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps11.jpg)  

10、设置字符集，从以下字符集列表中选择，UTF-8通用字符集  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps12.jpg)  

 

11、选择文件系统，并指定数据库文件位置，只需要修改下根盘符即可  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps13.jpg)  

 

12、取消注册到选项  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps14.jpg)  

 

13、默认即可，不启用恢复  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps15.jpg)  

 

14、对所有账号使用相同的口令，设置密码：Sysadmin123  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps16.jpg)  

15、确认信息无误，点击安装，进入安装界面，等待安装完成  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps17.jpg)  

## ***\*三、创建表空间和用户\****  

1、创建初始化脚本E:\app\init.bat  

```  
sqlplus / as sysdba @create_user.sql  
```  

2、创建数据库建库和表空间文件E:\app\create_user.sql  

***\*修改项：\****  

实例名称：orcltest  

表空间目录：数据库数据文件所在目录  

数据库账号：orcltest  

数据库密码：orcltest  

 ```  
create tablespace  orcltestdata datafile  'E:\app\Administrator\virtual\oradata\orcltest\orcltestdata01.dbf' size 1024m autoextend on next 10M maxsize unlimited;  
create tablespace  orcltestidx  datafile  'E:\app\Administrator\virtual\oradata\orcltest\orcltestidx01.dbf' size 1024m autoextend on next 10M maxsize unlimited;  
create tablespace  orcltestlog  datafile  'E:\app\Administrator\virtual\oradata\orcltest\orcltestlog01.dbf' size 1024m autoextend on next 10M maxsize unlimited;  
create  temporary tablespace  orcltesttemp  tempfile 'E:\app\Administrator\virtual\oradata\orcltest\orcltesttemp01.dbf' size 1024m autoextend on next 10M maxsize unlimited;  
create   user  orcltest   identified by  orcltest  default tablespace  orcltestdata  temporary  tablespace  orcltesttemp profile default;  

grant execute on  dbms_aq  to orcltest;  
grant execute on dbms_aqadm to orcltest;  
grant create session       to orcltest;  
grant CREATE SEQUENCE      to orcltest;  
grant CREATE TRIGGER       to orcltest;  
grant CREATE CLUSTER       to orcltest;  
grant CREATE PROCEDURE     to orcltest;  
grant CREATE TYPE          to orcltest;  
grant CREATE OPERATOR      to orcltest;  
grant CREATE TABLE         to orcltest;  
grant CREATE INDEXTYPE     to orcltest;  
GRANT CREATE VIEW          TO orcltest;  
GRANT CREATE SYNONYM       TO orcltest;  
GRANT CREATE JOB           TO orcltest;  
GRANT ALTER TABLESPACE     TO orcltest;  
grant unlimited tablespace to orcltest;  
                                    
GRANT DEBUG CONNECT SESSION TO orcltest;  
GRANT DEBUG any PROCEDURE   TO orcltest;  
                                    
GRANT select on dba_data_files TO orcltest;  
GRANT select on SM$TS_AVAIL    TO orcltest;  
GRANT select on SYS.SM$TS_USED TO orcltest;  
GRANT select on SYS.SM$TS_FREE TO orcltest;  
                                        
grant select on v_$sysstat to   orcltest;  
grant select on v_$statname to  orcltest;  
grant select on v_$session to   orcltest;  
grant select on v_$mystat to    orcltest;  
grant select on v_$sesstat to   orcltest;  
 ```  





 

3、执行初始化脚本E:\app\init.bat，即可完成表空间创建，并支持自动扩展以及数据库用户创建和授权  

## 四、listen和tns  

1、Listener  

默认listener安装完后都已经创建好，无需配置  

E:\app\Administrator\virtual\product\12.2.0\dbhome_1\network\admin\listener.ora  

```  
# listener.ora Network Configuration File: e:\app\Administrator\virtual\product\12.2.0\dbhome_1\network\admin\listener.ora  
# Generated by Oracle configuration tools.  

SID_LIST_LISTENER =  
  (SID_LIST =  
    (SID_DESC =  
      (SID_NAME = CLRExtProc)  
      (ORACLE_HOME = e:\app\Administrator\virtual\product\12.2.0\dbhome_1)  
      (PROGRAM = extproc)  
      (ENVS = "EXTPROC_DLLS=ONLY:e:\app\Administrator\virtual\product\12.2.0\dbhome_1\bin\oraclr12.dll")  
    )  
  )  

LISTENER =  
  (DESCRIPTION_LIST =  
    (DESCRIPTION =  
      (ADDRESS = (PROTOCOL = TCP)(HOST = WIN-JA5BT95357S)(PORT = 1521))  
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))  
    )  
  )  
```  

 

2、tns  

tns是客户端使用的，系统默认也创建好了  

E:\app\Administrator\virtual\product\12.2.0\dbhome_1\network\admin\tnsnames.ora  

```  
# tnsnames.ora Network Configuration File: e:\app\Administrator\virtual\product\12.2.0\dbhome_1\network\admin\tnsnames.ora  
# Generated by Oracle configuration tools.  

LISTENER_ORCLTEST =  
  (ADDRESS = (PROTOCOL = TCP)(HOST = WIN-JA5BT95357S)(PORT = 1521))  


ORACLR_CONNECTION_DATA =  
  (DESCRIPTION =  
    (ADDRESS_LIST =  
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))  
    )  
    (CONNECT_DATA =  
      (SID = CLRExtProc)  
      (PRESENTATION = RO)  
    )  
  )  

ORCLTEST =  
  (DESCRIPTION =  
    (ADDRESS = (PROTOCOL = TCP)(HOST = WIN-JA5BT95357S)(PORT = 1521))  
    (CONNECT_DATA =  
      (SERVER = DEDICATED)  
      (SERVICE_NAME = orcltest)  
    )  
  )  


```  



## ***\*五、测试连接\****  

cdm窗口下执行sqlplus orcltest/orcltest@ORCLTEST，提示成功登入便可成功  

![img](file:///C:\Users\guott\AppData\Local\Temp\ksohtml18152\wps18.jpg)  

## ***\*六、常用操作\****  

***\*管理数据库\****  

开始-所有应用--Oracle Administration Assistant for Windows工具实现对数据库启停  

 

***\*查看实例名称(sid)\****  

select instance_name from  V$instance;  

一般默认情况下sid与你的数据库的名称是一样的！  

 

***\*查看service_name\*******\*名称\****  

show parameter service;  

 

***\*查看表空间位置及大小\****  

SELECT tablespace_name,  
file_id,  
file_name,  
round(bytes / (1024 * 1024), 0) total_space  
FROM dba_data_files  
ORDER BY tablespace_name;  