在CentOS上安装MongoDB社区版   
https://docs.mongodb.com/master/tutorial/install-mongodb-on-red-hat/

# 1. 配置程序包yum源
创建/etc/yum.repos.d/mongodb-org-4.2.repo文件  

> [mongodb-org-4.2]  
> name=MongoDB Repository  
> baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/  
> gpgcheck=1  
> enabled=1  
> gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc  


# 2. 安装MongoDB软件包

`sudo yum install -y mongodb-org-4.2.2 mongodb-org-server-4.2.2 mongodb-org-shell-4.2.2 mongodb-org-mongos-4.2.2 mongodb-org-tools-4.2.2`  


# 3. 运行MongoDB社区版

**默认目录**  
默认情况下，MongoDB使用mongod用户帐户运行，并使用以下默认目录：  
/var/lib/mongo (the data directory)  
/var/log/mongodb (the log directory)  

**配置SELinux**  

## 启动MongoDB  
`sudo service mongod start`  
`sudo chkconfig mongod on`

## 验证MongoDB已成功启动
检查/var/log/mongodb/mongod.log中日志文件的内容来验证mongod进程是否已成功启动  

> [initandlisten] waiting for connections on port <port>

其中\<port\>是/etc/mongod.conf中配置的端口，默认情况下为27017  

## 停止MongoDB
`sudo service mongod stop`

## 重新启动MongoDB
`sudo service mongod restart`

## 开始使用MongoDB
在与mongod相同的主机上启动mongo shell。可以在没有任何命令行选项的情况下运行mongo shell，使用默认端口27017连接到在本地主机上运行的mongod：  
`mongo`





