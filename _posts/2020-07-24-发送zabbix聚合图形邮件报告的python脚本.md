#发送zabbix聚合图形邮件报告的python脚本


```
#!/usr/bin/python
# coding:utf-8

# zabbix-screen.py

import datetime
import os
import time
import argparse
import tempfile


# zabbix api
# pip install pyzabbix
from pyzabbix import ZabbixAPI, ZabbixAPIException

# 爬虫
# pip install bs4
import requests
import re
from bs4 import BeautifulSoup

# 邮件
import smtplib
from email.header import Header
from email.mime.text import MIMEText
from email.mime.image import MIMEImage
from email.mime.multipart import MIMEMultipart


class zabbixreport:
    def __init__(self, zabbixapi):
        self.zapi = zabbixapi
        self.__server = zapi.url.replace('api_jsonrpc.php', '')
        self.__sessionid = zapi.check_authentication()["sessionid"]
        self.__path = tempfile.gettempdir()

        self.dtfrom = ''
        self.dtto = ''
        self.threadmax = 1

        self.smtp_server = ''
        self.smtp_port = '25'
        self.sender_name = ''
        self.sender_email = ''
        self.sender_password = ''
        self.receiver = []
        self.subject = ''

    def get_screenname(self, screenid):
        screen = zapi.screen.get(screenids=screenid, output="extend")
        result = screen[0]['name']
        return result

    # 获得包含graphid等的列表
    def get_graphids(self, screenid):
        # ['screenid','graphid', 'width', 'height', 'x', 'y','0.png']
        result = []
        screenitems = zapi.screenitem.get(screenids=screenid, output="extend", sortfield='screenitemid')
        i = 0
        for screenitem in screenitems:
            item = []
            item.append(screenitem['screenid'])
            item.append(screenitem['resourceid'])
            item.append(screenitem['width'])
            item.append(screenitem['height'])
            item.append(screenitem['x'])
            item.append(screenitem['y'])
            item.append('{}.png'.format(i))
            result.append(item)
            i = i + 1
        return result

    # 下载图片
    def multhr_get_graphs(self, *args):
        pass

    def get_graphfile(self, graphid):
        # chart2.php?screenid={}&graphid={}&width={}&height={}&profileIdx=web.screens.filter&from=now-1d%2Fd&to=now-1d%2Fd
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
        cookie['zbx_sessionid'] = self.__sessionid
        # url
        url = "{}chart2.php".format(self.__server)
        # 参数
        param = {}
        param['profileIdx'] = "web.screens.filter"
        param['from'] = self.dtfrom
        param['to'] = self.dtto
        param['screenid'] = graphid[0]
        param['graphid'] = graphid[1]
        param['width'] = graphid[2]
        param['height'] = graphid[3]
        outpath = os.path.join(self.__path, graphid[0])
        if not os.path.exists(outpath):
            os.makedirs(outpath)
        try:
            # 发送请求
            response = requests.get(
                url, params=param, headers=header, cookies=cookie)
            # 如果响应状态码不是200，就抛出异常
            response.raise_for_status()
        except requests.RequestException as e:
            print(e)
        else:
            # 获得响应数据
            data = response.content
            # 写入图片文件
            try:
                filepath = os.path.join(outpath, graphid[6])
                with open(filepath, "wb") as f:
                    f.write(data)
                result = [f.name, graphid[6]]
                return result
            except Exception as e:
                print(e)
        # print(graphid)
        # time.sleep(1)

    def get_message(self, screensid, graphfiles):
        # url
        url = "{}screens.php".format(self.__server)
        # print(url)
        # 参数
        param = {}
        param['elementid'] = screensid
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
        cookie['zbx_sessionid'] = self.__sessionid
        try:
            # 发送请求
            response = requests.get(
                url, params=param, headers=header, cookies=cookie)
            # 如果响应状态码不是200，就抛出异常
            response.raise_for_status()
        except requests.RequestException as e:
            print(e)
        else:
            # 获得响应数据
            str_html = response.text

            # 设置总的邮件体对象，对象类型为mixed
            msg_root = MIMEMultipart('mixed')

            # 邮件添加的头尾信息等
            self.subject = self.get_screenname(screensid)
            msg_root['From'] = Header('%s<%s>' % (
                self.sender_name, self.sender_email), 'utf-8')
            msg_root['To'] = ','.join(self.receiver)
            msg_root['subject'] = Header(self.subject, 'utf-8')

            # 构造超文本
            soup = BeautifulSoup(str_html, 'html.parser')
            html_info = soup.select('body > main > div.table-forms-container')
            # 修改 a标签 为 img标签
            for h in html_info:
                i = 0
                for link in h.find_all('a'):
                    imgtag = soup.new_tag('img')
                    imgtag['src'] = 'cid:{}.png'.format(i)
                    link.replaceWith(imgtag)
                    i = i + 1
            html_sub = MIMEText(str(html_info[0]), 'html', 'utf-8')
            # 把构造的内容写到邮件体中
            msg_root.attach(html_sub)

            # 构造图片
            for gf in graphfiles:
                image_file = open(gf[0], 'rb').read()
                image = MIMEImage(image_file)
                image.add_header('Content-ID', '<{}>'.format(gf[1]))
                # 如果不加下边这行代码的话，会在收件方显示乱码的bin文件，下载之后也不能正常打开
                image["Content-Disposition"] = 'attachment; filename="{}"'.format(
                    gf[1])
                msg_root.attach(image)

            # 构造附件
            # txt_file = open(r'hello_world.txt', 'rb').read()
            # txt = MIMEText(txt_file, 'base64', 'utf-8')
            # txt["Content-Type"] = 'application/octet-stream'
            # #以下代码可以重命名附件为hello_world.txt
            # txt.add_header('Content-Disposition', 'attachment', filename='hello_world.txt')
            # msg_root.attach(txt)

            return msg_root

    def send_mail(self, message_obj):
        try:
            smtp_obj = smtplib.SMTP(self.smtp_server, self.smtp_port)
            smtp_obj.login(self.sender_email, self.sender_password)
            smtp_obj.sendmail(self.sender_email,
                              self.receiver, message_obj.as_string())
            smtp_obj.quit()
            print('sendemail successful!')
        except Exception as e:
            print('sendemail failed next is the reason')
            print(e)


if __name__ == "__main__":
    # 聚合图形id
    screensids = [93, 95]

    # # 参数
    # parser = argparse.ArgumentParser()
    # parser.add_argument('-id', '--screen_id',  action="store",
    #                     help='The screen id')
    # args = parser.parse_args()

    # 信息
    server = 'http://localhsot/zabbix'
    username = 'Admin'
    password = 'zabbix'
    
    today = datetime.date.today()
    yesterday = today - datetime.timedelta(days=1)
    now = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    todaystr = today.strftime('%Y-%m-%d %H:%M:%S')
    dtfrom = datetime.datetime.strptime(todaystr, '%Y-%m-%d %H:%M:%S') - datetime.timedelta(hours=16)
    dtto = datetime.datetime.strptime(todaystr, '%Y-%m-%d %H:%M:%S') + datetime.timedelta(hours=8)

    # 登入
    zapi = ZabbixAPI(server)
    try:
        zapi.login(username, password)
    except ZabbixAPIException as e:
        print(e)
    else:
        zr = zabbixreport(zapi)
        zr.dtfrom = dtfrom
        zr.dtto = dtto

        zr.smtp_server = 'localhsot'
        zr.sender_name = 'zabbix'
        zr.sender_email = 'zabbix@mail.com'
        zr.sender_password = 'zabbix'
        zr.receiver = ['user@mail.com']

        for screensid in screensids:
            # 获取聚合图形项
            graphids = zr.get_graphids(screensid)

            # 下载图片
            graphfiles = []
            for graphid in graphids:
                gf = zr.get_graphfile(graphid)
                graphfiles.append(gf)

            # 构造邮件
            msgobj = zr.get_message(screensid, graphfiles)

            # 发送邮件
            zr.send_mail(msgobj)

        print("Done.")
```
