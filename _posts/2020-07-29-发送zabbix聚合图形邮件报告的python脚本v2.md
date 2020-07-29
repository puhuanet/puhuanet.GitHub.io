# 发送zabbix聚合图形邮件报告的python脚本v2


## 如何使用
```
usage: zabbix_screen.py [-h] [-sids SCREENIDS] [-hids HOSTIDS] [-r RECEIVERS]

optional arguments:
  -h, --help            show this help message and exit
  -sids SCREENIDS, --screenids SCREENIDS
                        The Screen IDs
  -hids HOSTIDS, --hostids HOSTIDS
                        The Host IDs
  -r RECEIVERS, --receivers RECEIVERS
                        The receiver mail address
```

## zabbix_screen.py
```
#!/usr/bin/python
# coding:utf-8

import argparse
import datetime

from ZbxReport import ZbxReport

if __name__ == "__main__":
    # 如果脚本未获取到外部参数，则使用脚本内部设定的参数
    screenids = ['93', '95']
    hostids = []
    receivers = ['jacky@puhua.net']

    # 获取外部参数
    parser = argparse.ArgumentParser()
    parser.add_argument('-sids', '--screenids', action="store",
                        help='The Screen IDs')
    parser.add_argument('-hids', '--hostids', action="store",
                        help='The Host IDs')
    parser.add_argument('-r', '--receivers', action="store",
                        help='The receiver mail address')
    args = parser.parse_args()

    # 如果脚本未获取到外部参数，则使用脚本内部设定的参数
    if args.screenids:
        screenids = args.screenids.split(',')
    if args.hostids:
        hostids = args.hostids.split(',')
    if args.receivers:
        receivers = args.receivers.split(',')

    # 信息
    server = 'http://localhost/zabbix'
    username = 'Admin'
    password = 'zabbix'

    today = datetime.date.today()
    yesterday = today - datetime.timedelta(days=1)
    now = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    today_str = today.strftime('%Y-%m-%d %H:%M:%S')
    dtfrom = datetime.datetime.strptime(today_str, '%Y-%m-%d %H:%M:%S') - datetime.timedelta(hours=16)
    dtto = datetime.datetime.strptime(today_str, '%Y-%m-%d %H:%M:%S') + datetime.timedelta(hours=8)

    # 登入
    zr = ZbxReport(server)
    try:
        zr.login(username, password)
    except ZbxReport as e:
        print(e)
    else:
        zr.dtfrom = dtfrom
        zr.dtto = dtto

        zr.smtp_server = 'smtp.puhua.net'
        zr.sender_name = 'zabbix'
        zr.sender_email = 'zabbix@puhua.net'
        zr.sender_password = 'password'
        zr.receivers = receivers

        zr.screenids = screenids
        zr.hostids = hostids

        zr.send_report()

        print("Done.")

```

## ZbxReport.py
```
#!/usr/bin/python
# coding:utf-8

# pip install bs4
# pip install pyzabbix

import os
import re
import smtplib
import tempfile
import threading
import time
from email.header import Header
from email.mime.image import MIMEImage
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

import requests
from jinja2 import Template
from pyzabbix import ZabbixAPI, ZabbixAPIException


class ZbxReport(ZabbixAPI, ZabbixAPIException):
    def __init__(self,
                 server='http://localhost/zabbix',
                 session=None,
                 use_authenticate=False,
                 timeout=None
                 ):
        """
        Parameters:
            server: Base URI for zabbix web interface (omitting /api_jsonrpc.php)
            session: optional pre-configured requests.Session instance
            use_authenticate: Use old (Zabbix 1.8) style authentication
            timeout: optional connect and read timeout in seconds, default: None
                     (if you're using Requests >= 2.4 you can set it as tuple: "(connect, read)"
                      which is used to set individual connect and read timeouts.)
        """
        if server.endswith('/'):
            server = server.rstrip('/')

        super(ZbxReport, self).__init__(server, session, use_authenticate, timeout)

        # self.__sessionid = self.check_authentication()["sessionid"]
        self.baseuri = server
        self.__path = tempfile.gettempdir()

        self.dtfrom = ''
        self.dtto = ''
        self.smtp_server = ''
        self.smtp_port = '25'
        self.sender_name = ''
        self.sender_email = ''
        self.sender_password = ''
        self.receivers = []
        self.subject = ''

        self.__screenname = ''
        self.__hostname = ''
        self.__graphids = []
        self.__graphfiles = []

    def send_report(self):
        screenids = self.screenids
        hostids = self.hostids
        # 通过聚合图形id和主机id 获取聚合图形项 graphids
        if len(screenids) == 0 and len(hostids) == 0:
            print('请输入参数')
            exit()
        elif len(screenids) > 0 and len(hostids) == 0:
            for screenid in screenids:
                if self.get_graphids(self.get_graphs, screenid):
                    msg = self.get_message(screenid)
                    self.send_mail(msg)
        elif len(screenids) == 0 and len(hostids) > 0:
            for hostid in hostids:
                if self.get_graphids(self.get_graphs, hostid):
                    msg = self.get_message(hostid)
                    self.send_mail(msg)
        else:
            for screenid in screenids:
                for hostid in hostids:
                    if self.get_graphids(self.get_graphs, screenid, hostid):
                        msg = self.get_message(screenid, hostid)
                        self.send_mail(msg)

    # 获得包含graphid graphuri等的列表 (方法二 通过爬虫)
    # 通过回调函数下载图片，并返回图片路径列表
    def get_graphids(self, callback, screenid='0', hostid='0'):
        # screens.php?config=screens.php&elementid={}&hostid={}&from={}&to={}
        self.__graphids = []
        self.__graphfiles = []
        self.__screenname = ''
        self.__hostname = ''

        if screenid != '0':
            try:
                self.__screenname = self.screen.get(screenids=screenid, output=['name'])[0]['name']
            except Exception as e:
                print('screenid:%s not found' % (screenid))
                return False

        if hostid != '0':
            try:
                self.__hostname = self.host.get(hostids=hostid, output=['name'])[0]['name']
            except Exception as e:
                print('hostid:%s not found' % (hostid))
                return False

        url = '{}/screens.php'.format(self.baseuri)
        param = {}
        param['config'] = 'screens.php'
        param['elementid'] = screenid
        param['hostid'] = hostid
        param['from'] = self.dtfrom
        param['to'] = self.dtto
        header = {}
        header['Connection'] = 'Keep-Alive'
        header[
            'User-Agent'] = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/45.0.2454.101 Safari/537.36'
        header['Cache-Control'] = 'no-cache'
        header['Accept-Language'] = 'zh-CN'
        header['Accept-Encoding'] = 'gzip'
        cookie = {}
        cookie['zbx_sessionid'] = self.check_authentication()["sessionid"]
        # print('{}?{}'.format(url, urlencode(param)))
        response = requests.get(url, params=param, headers=header, cookies=cookie)
        rc = re.compile('(chart\d?\.php\?graphid=(\d+)[^"]+)')
        graphids = rc.findall(response.text)

        for graphid in graphids:
            filename = '%s.png' % (graphid[1])
            self.__graphids.append(graphid[1])
            self.__graphfiles.append(os.path.join(self.__path, screenid, hostid, filename))
        callback(graphids, screenid, hostid)
        return graphids

    # 多线程获取图片
    def get_graphs(self, graphids, screenid='0', hostid='0'):
        thrsem = threading.Semaphore(10)
        ts = []
        start_time = time.time()
        with thrsem:
            for graphid in graphids:
                tt = threading.Thread(target=self.get_graphfile, args=(graphid, screenid, hostid))
                ts.append(tt)
                tt.start()
            for t in ts:
                t.join()
        end_time = time.time()
        # print(str(end_time - start_time))

    # 下载图片
    def get_graphfile(self, graphid, screenid='0', hostid='0'):
        guri = graphid[0]
        gid = graphid[1]

        # 设置header
        header = {}
        header['Connection'] = 'Keep-Alive'
        header[
            'User-Agent'] = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/45.0.2454.101 Safari/537.36'
        header['Cache-Control'] = 'no-cache'
        header['Accept-Language'] = 'zh-CN'
        header['Accept-Encoding'] = 'gzip'
        # 设置cookies
        cookie = {}
        cookie['zbx_sessionid'] = self.check_authentication()["sessionid"]
        # url
        url = '{}/{}'.format(self.baseuri, guri)
        # print(url)
        outpath = os.path.join(self.__path, screenid, hostid)
        try:
            if not os.path.exists(outpath):
                os.makedirs(outpath)
        except OSError:
            pass
        try:
            # 发送请求
            response = requests.get(
                url, headers=header, cookies=cookie)
            # 如果响应状态码不是200，就抛出异常
            response.raise_for_status()
        except requests.RequestException as e:
            print(e)
        else:
            # 获得响应数据
            data = response.content
            # 写入图片文件
            try:
                filename = '%s.png' % (gid)
                filepath = os.path.join(outpath, filename)
                with open(filepath, "wb") as f:
                    f.write(data)
            except Exception as e:
                print(e)

    def get_message(self, screenid='0', hostid='0'):
        graphids = self.__graphids
        graphfiles = self.__graphfiles
        screenname = self.__screenname
        hostname = self.__hostname

        # 设置总的邮件体对象，对象类型为mixed
        msg = MIMEMultipart('mixed')

        # 邮件添加的头尾信息等
        self.subject = '%s%s' % (hostname, screenname)
        msg['From'] = Header('%s<%s>' % (
            self.sender_name, self.sender_email), 'utf-8')
        msg['To'] = ','.join(self.receivers)
        msg['subject'] = Header(self.subject, 'utf-8')

        # 构造超文本
        template_str = u'''<!DOCTYPE html>
                            <html>
                                <head>
                                    <meta charset="utf-8">
                                    <title></title>
                                </head>
                                <body>
                                    <table>
                                        {% for gid in graphids %}
                                            {%- if loop.first -%}
                                                <tr>
                                            {%- endif %}
                                            <td><img src="cid:{{gid}}.png"></td>
                                            {% if loop.index0 is odd -%}
                                                </tr>
                                                <tr> 
                                            {%- endif -%}
                                            {%- if loop.last %}
                                                </tr>
                                            {%- endif -%}
                                        {%- endfor %}
                                    </table>
                                </body>
                            </html>'''
        template = Template(template_str)
        render_content = template.render(graphids=graphids)
        html_sub = MIMEText(render_content, 'html', 'utf-8')
        msg.attach(html_sub)

        # 构造图片
        for gf in graphfiles:
            image_file = open(gf, 'rb').read()
            filename = os.path.basename(gf)
            image = MIMEImage(image_file)
            image.add_header('Content-ID', '<{}>'.format(filename))
            # 如果不加下边这行代码的话，会在收件方显示乱码的bin文件，下载之后也不能正常打开
            image["Content-Disposition"] = 'attachment; filename="{}"'.format(filename)
            msg.attach(image)

        # 构造附件
        # txt_file = open(r'D:\python_files\files\hello_world.txt', 'rb').read()
        # txt = MIMEText(txt_file, 'base64', 'utf-8')
        # txt["Content-Type"] = 'application/octet-stream'
        # #以下代码可以重命名附件为hello_world.txt
        # txt.add_header('Content-Disposition', 'attachment', filename='hello_world.txt')
        # msg.attach(txt)

        return msg

    def send_mail(self, message_obj):
        try:
            smtp_obj = smtplib.SMTP(self.smtp_server, self.smtp_port)
            smtp_obj.login(self.sender_email, self.sender_password)
            smtp_obj.sendmail(self.sender_email,
                              self.receivers, message_obj.as_string())
            smtp_obj.quit()
            print('%s sendemail successful' % (self.subject))
        except Exception as e:
            print('%s sendemail failed' % s(self.subject))
            print(e)

```