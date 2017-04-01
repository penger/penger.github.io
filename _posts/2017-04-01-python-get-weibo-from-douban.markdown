---
layout: post
category: "read"
title:  "python 爬取豆瓣微博"
tags: [python]
---

## 场景
python 下载验证码图片,并且爬取豆瓣微博
###  
```
# -*-coding:UTF-8-*-
import cookielib
import os
import re
import sys
import urllib
import urllib2
import time
import random

from bs4 import BeautifulSoup

reload(sys)
sys.setdefaultencoding('utf-8')
temp_path = "info_niujishou.txt"
if os.path.exists(temp_path):
    os.remove(temp_path)


info_file = open(temp_path, 'w')

loginurl = 'https://www.douban.com/accounts/login'
cookie = cookielib.CookieJar()
opener = urllib2.build_opener(urllib2.HTTPCookieProcessor(cookie))
params = {
    "form_email": "username@sina.com",
    "form_password": "password",
    "source": "index_nav" # 没有的话登录不成功
}
# 从首页提交登录
response=opener.open(loginurl, urllib.urlencode(params))

print response.geturl()

# 验证成功跳转至登录页
if response.geturl() == "https://www.douban.com/accounts/login":
    html=response.read()
    # 验证码图片地址
    imgurl=re.search('<img id="captcha_image" src="(.+?)" alt="captcha" class="captcha_image"/>', html)
    if imgurl:
        url=imgurl.group(1)
        # 将图片保存至同目录下
        if os.path.exists('v.jpg'):
            os.remove('v.jpg')
        res=urllib.urlretrieve(url, 'v.jpg')
        # 获取captcha-id参数
        captcha=re.search('<input type="hidden" name="captcha-id" value="(.+?)"/>' ,html)
        if captcha:
            print "图片地址为: ",captcha
            vcode=raw_input('请输入图片上的验证码：')
            params["captcha-solution"] = vcode
            params["captcha-id"] = captcha.group(1)
            params["user_login"] = "登录"
            # 提交验证码验证
            response=opener.open(loginurl, urllib.urlencode(params))
            ''' 登录成功跳转至首页 '''
            print response.geturl() , "--------------------------"
            if response.geturl() == "https://www.douban.com/":
                print response.read()
                print 'login success ! '
                print '准备进行爬虫'
elif response.geturl()=='https://www.douban.com/':
    print 'login success without captcha '
else:
    print 'unknown error for return url is :%s' % response.geturl()
for i in range(1, 526):
    time.sleep(random.randint(5, 10))
    print "爬出第%d 篇广播" % i
    info_file.write("爬出第%d 篇广播\n" % i)
    new_url = 'https://www.douban.com/people/nimamui/statuses?p='+bytes(i)
    content=opener.open(new_url).read()
    soup = BeautifulSoup(content, "html.parser")
    weibo_content=soup.find(class_='grid-16-8 clearfix').find_all(class_='bd')
    count =0
    for wc in weibo_content:
        # if count == 0:
        # print i
        print '--------------------------%d' % count
        if wc.blockquote:
            say = str(wc.blockquote.p.string).replace(' ', '').strip()
            print say
            info_file.write(say+'\n')
        for line in str(wc).split("\n"):
            if line.count('title'):
                # print line
                m = re.match('<span class="created_at" title="([^"]+)".*',line)
                if m:
                    time_time = m.group(1)
                    info_file.write(time_time)
                    print time_time
info_file.close()

```