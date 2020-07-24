在CentOS 7上安装Graylog   
https://docs.graylog.org/en/3.1/pages/installation/os/centos.html   

# 准备安装环境

**在安装Graylog服务之前，请确保安装和配置以下软件：**
* Java ( >= 8 )
* Elasticsearch (5.x or 6.x)
* MongoDB (3.6 or 4.0)

**警告**  
* Graylog 3不适用于Elasticsearch 7.x  
* Graylog 3不适用于MongoDB 4.2  

## 安装java jdk
`$ sudo yum install java-1.8.0-openjdk-headless.x86_64`

## 安装pwgen(密码生成器)
`$ sudo yum install epel-releasesudo | yum install pwgen`

## 安装MongoDB社区版

参考:[在centos上安装mongodb社区版](https://www.puhua.net/blog/posts/2019/12/14/%E5%9C%A8CentOS%E4%B8%8A%E5%AE%89%E8%A3%85MongoDB%E7%A4%BE%E5%8C%BA%E7%89%88.html)

添加/etc/yum.repos.d/mongodb-org.repo  
```
[mongodb-org-4.0]  
name=MongoDB Repository  
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/  
gpgcheck=1  
enabled=1  
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc   
```

安装mogodb
`sudo yum install mongodb-org`

设置启动
```
sudo systemctl daemon-reload  
sudo systemctl enable mongod.service  
sudo systemctl start mongod.service   
```  
 

## 安装Elasticsearch
参考：[Elasticsearch安装指南](https://www.puhua.net/blog/posts/2019/12/15/Elasticsearch%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97.html)

安装Elastic GPG key  
`rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch`  

添加/etc/yum.repos.d/elasticsearch.repo   
```
[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc
```

安装Elasticsearch  
`sudo yum install elasticsearch-oss`

修改Elasticsearch配置文件/etc/elasticsearch/elasticsearch.yml
```
cluster.name: graylog
action.auto_create_index: false
```

启动Ealsticsearch
```
$ sudo systemctl daemon-reload
$ sudo systemctl enable elasticsearch.service
$ sudo systemctl restart elasticsearch.service
```

# 安装Graylog

安装Graylog存储库配置和Graylog
```
$ sudo rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-3.1-repository_latest.rpm
$ sudo yum install graylog-server
```

安装插件
```
sudo yum install graylog-server graylog-enterprise-plugins graylog-integrations-plugins graylog-enterprise-integrations-plugins
```

按照/etc/graylog/server/server.conf中的说明进行操作，并添加password_secret和root_password_sha2

创建您的root_password_sha2   
`echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1`

设置启动
```
$ sudo systemctl daemon-reload
$ sudo systemctl enable graylog-server.service
$ sudo systemctl start graylog-server.service
```
