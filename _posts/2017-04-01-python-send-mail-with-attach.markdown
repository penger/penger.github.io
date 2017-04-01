
---
layout: post
category: "read"
title:  "python 发送带附件的邮件"
tags: [python]
---

## 场景
脚本运行完成需要将日志以附件的形式发送到邮箱
###  
```
#!/usr/bin/python
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import smtplib
msg = MIMEMultipart()
att1 = MIMEText(open('/home/bigdatadev/file', 'rb').read(), 'base64', 'gb2312')
att1["Content-Type"] = 'application/octet-stream'
att1["Content-Disposition"] = 'attachment; filename="test"'
msg.attach(att1)
msg['to'] = 'from@bl.com'
msg['from'] = 'to@bl.com'
msg['subject'] = 'hello world'
try:
    server = smtplib.SMTP()
    server.connect('mail.bl.com')
    server.login('username','password')
    server.sendmail(msg['from'], msg['to'],msg.as_string())
    server.quit()
    print '....'
except Exception, e:  
    print str(e)
```