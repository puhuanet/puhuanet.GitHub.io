# OwnCloud配置外部存储及部署Collabora服务器

## OwnCloud配置外部存储

https://doc.owncloud.com/server/admin_manual/configuration/files/external_storage/smb.html

### 查看已启用的模块
/opt/owncloud-10.4.1-2/php/bin/php -m | grep smbclient

### 准备
```
sudo yum groups install "Development Tools"
sudo yum install -y autoconf
sudo yum install -y libsmbclient-devel
```

### 下载、解压
```
cd /tmp
wget https://pecl.php.net/get/smbclient-1.0.0.tgz
tar xzf smbclient-1.0.0.tgz
```

### 用php自带的phpize生成configure文件
```
cd smbclient-1.0.0.tgz
/opt/owncloud-10.4.1-2/php/bin/phpize
```

### 编译
```
./configure --with-php-config=/opt/owncloud-10.4.1-2/php/bin/php-config
sudo make && make install
Installing shared extensions:     /opt/owncloud-10.4.1-2/php/lib/php/extensions/
```

### 启用模块
```
vi /opt/owncloud-10.4.1-2/php/etc/php.ini
	extension=smbclient
```

### 查看已启用的模块
```
/opt/owncloud-10.4.1-2/php/bin/php -m | grep smbclient
```

### 重启服务
```
sudo /opt/owncloud-10.4.1-2/ctlscript.sh restart
```



## 部署Collabora服务器

https://github.com/owncloud/richdocuments

### 安装docker
```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce
sudo systemctl start docker
sudo systemctl enable docker
```

### 部署collabora服务器
```
docker run -t -d -p 9980:9980 -e "username=admin" -e "password=admin" --name collabora --cap-add MKNOD collabora/code
```

### Update Collabora Server docker and modify SSL settings
```
docker exec -u root -it collabora /bin/bash -c "apt-get -y update && apt-get -y install xmlstarlet && xmlstarlet ed --inplace -u \"/config/ssl/enable\" -v false /etc/loolwsd/loolwsd.xml && xmlstarlet ed --inplace -u \"/config/ssl/termination\" -v false /etc/loolwsd/loolwsd.xml"
docker restart collabora
```

```
vim /etc/loolwsd/loolwsd.xml

  <ssl desc="SSL settings">
    <enable type="bool" desc="Controls whether SSL encryption between browser and loolwsd is enabled (do not disable for production deployment). If default is false, must first be compiled with SSL support to enable." default="true">false</enable>
    <termination desc="Connection via proxy where loolwsd acts as working via https, but actually uses http." type="bool" default="true">false</termination>
    <cert_file_path desc="Path to the cert file" relative="false">/etc/loolwsd/cert.pem</cert_file_path>
    <key_file_path desc="Path to the key file" relative="false">/etc/loolwsd/key.pem</key_file_path>
    <ca_file_path desc="Path to the ca file" relative="false">/etc/loolwsd/ca-chain.cert.pem</ca_file_path>
    <cipher_list desc="List of OpenSSL ciphers to accept" default="ALL:!ADH:!LOW:!EXP:!MD5:@STRENGTH"/>
    <hpkp desc="Enable HTTP Public key pinning" enable="false" report_only="false">
      <max_age desc="HPKP's max-age directive - time in seconds browser should remember the pins" enable="true">1000</max_age>
      <report_uri desc="HPKP's report-uri directive - pin validation failure are reported at this URL" enable="false"/>
      <pins desc="Base64 encoded SPKI fingerprints of keys to be pinned">
        <pin/>
      </pins>
    </hpkp>
  </ssl>
```

### 设定
Access Collabora Admin at http://[your-host-public-ip]:9980/loleaflet/dist/admin/admin.html 
e.g. 172.16.12.95, and set in Settings -> Admin -> Additional -> Collabora Online server -> http://[your-host-public-ip]:9980

