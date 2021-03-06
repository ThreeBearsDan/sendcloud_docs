
## WEBAPI 普通发送
```
#coding:utf-8                                                                   

import requests                                                                 

url = "http://sendcloud.sohu.com/webapi/mail.send.json"                         

API_USER = '...'
API_KEY = '...'

params = {                                                                      
    "api_user": API_USER, # 使用api_user和api_key进行验证                       
    "api_key" : API_KEY,                                             
    "to" : "to1@domain.com;to2@domain.com", # 收件人地址, 用正确邮件地址替代, 多个地址用';'分隔  
    "from" : "sendcloud@sendcloud.org", # 发信人, 用正确邮件地址替代     
    "fromname" : "SendCloud",                                                    
    "subject" : "SendCloud python common",                              
    "html": "欢迎使用SendCloud",
    "resp_email_id": "true",
}                                                                               

filename = "/path/file.pdf"
display_filename = "attachment.pdf"

files = { "file1" : (display_filename, open(filename,"rb"))}

r = requests.post(url, files=files, data=params)

print r.text

```
[downloads](../downloads/code/python/python_common.py)

- - - 

## WEBAPI 模板发送
```
#coding:utf-8                                                                   

import requests, json                                                                

url = "http://sendcloud.sohu.com/webapi/mail.send_template.json"                         

API_USER = '...'
API_KEY = '...'

sub_vars = {
    'to': ['to1@domain.com', 'to2@domain.com'],
    'sub': {
        '%name%': ['user1', 'user2'],
        '%money%': ['1000', '2000'],
    }
}

params = {
    "api_user": API_USER, # 使用api_user和api_key进行验证                       
    "api_key" : API_KEY,                                             
    "template_invoke_name" : "test_template",
    "substitution_vars" : json.dumps(sub_vars),
    "from" : "sendcloud@sendcloud.org", # 发信人, 用正确邮件地址替代
    "fromname" : "SendCloud",
    "subject" : "SendCloud python template",
    "resp_email_id": "true",
}

filename = "..."
display_filename = "..."

files = { "file1" : (display_filename, open(filename,"rb"))}

r = requests.post(url, files=files, data=params)

print r.text

```
[downloads](../downloads/code/python/python_template.py)

- - - 

## WEBAPI 模板 && 地址列表 发送
```
#coding:utf-8                                                                   

import requests                                                                 

url = "http://sendcloud.sohu.com/webapi/mail.send_template.json"                         

API_USER = '...'
API_KEY = '...'

params = {                                                                      
    "api_user": API_USER, # 使用api_user和api_key进行验证                       
    "api_key" : API_KEY,                                             
    "to" : "test@maillist.sendcloud.org", # 使用地址列表的别称地址
    "from" : "sendcloud@sendcloud.org", # 发信人, 用正确邮件地址替代
    "fromname" : "SendCloud",                                                    
    "subject" : "SendCloud python template address_list",                              
    "template_invoke_name" : "sendcloud_template",
    "use_maillist" : "true",
    "resp_email_id": "true",
}                                                                               

filename = "/path/file.pdf"
display_filename = "attachment.pdf"

files = { "file1" : (display_filename, open(filename,"rb"))}

r = requests.post(url, files=files, data=params)

print r.text
```
[downloads](../downloads/code/python/python_template_maillist.py)
- - - 

## SMTP 方式发送
```
# -*- coding:utf-8 -*-                                                          

from smtplib import SMTP
from email.mime.multipart import MIMEMultipart
from email.header import Header
from email.mime.base import MIMEBase
from email.mime.text import MIMEText
from email.utils import COMMASPACE, formatdate, formataddr
from email import encoders

import base64, time, simplejson, os, json
from datetime import datetime, date

HOST = 'smtpcloud.sohu.com'
PORT = 25
API_USER = '...'
API_KEY = '...'

DEBUG_MODE = False
USE_SSL = False

def _message_id(reply):
    message_id = None

    if str(reply[0]) == '250': 
        message_id = str(reply[1]).split('#')[1]

    return message_id

def send(mail_from, ffrom, rcpt_tos, reply_to, subject, content, files):                                                                  
    ret = {}

    s = SMTP('%s:%d' % (HOST, PORT))
    s.set_debuglevel(DEBUG_MODE)
    if USE_SSL:
        s.starttls()
    s.login(API_USER, API_KEY)

    for rcpt_to in rcpt_tos:
        ret[rcpt_to] = None

        msg = MIMEMultipart('alternative')
        msg['subject'] = subject
        msg['from'] = ffrom
        msg['reply-to'] = reply_to
        msg['to'] = rcpt_to

        part = MIMEText(content, 'html', 'utf8')
        msg.attach(part)

        for f in files:
            part = MIMEBase('application', 'octet-stream') #'octet-stream': binary data 
            part.set_payload(open(f, 'rb').read())
            encoders.encode_base64(part)

            # 如果附件名称含有中文, 则 filename 要转换为gb2312编码, 否则就会出现乱码
            # unicode转换方法:  basename.encode('gb2312')  
            # utf-8转换方法:    basename.decode('utf-8').encode('gb2312')  
            filename = os.path.basename(f)
            part.add_header('Content-Disposition', 'attachment; filename="%s"' % filename.decode('utf8').encode('gb2312'))
            msg.attach(part)

        s.mail(mail_from)
        s.rcpt(rcpt_to)
        reply_data = s.data(msg.as_string())
        s.rset()

        message_id = _message_id(reply_data)
        ret[rcpt_to] = message_id + '0$' + rcpt_to

    s.quit()

    print ret
    return ret

def sendn(mail_from, ffrom, x_smtpapi, reply_to, subject, content, files): 
    ret = {}

    s = SMTP('%s:%d' % (HOST, PORT))
    s.set_debuglevel(DEBUG_MODE)
    if USE_SSL:
        s.starttls()
    s.login(API_USER, API_KEY)

    msg = MIMEMultipart('alternative')
    msg['subject'] = subject
    msg['from'] = ffrom
    msg['reply-to'] = reply_to
    msg['X-SMTPAPI'] = Header(base64.b64encode(simplejson.dumps(x_smtpapi)))

    part = MIMEText(content, 'html', 'utf8')
    msg.attach(part)

    for f in files:
        part = MIMEBase('application', 'octet-stream') #'octet-stream': binary data 
        part.set_payload(open(f, 'rb').read())
        encoders.encode_base64(part)

        # 如果附件名称含有中文, 则 filename 要转换为gb2312编码, 否则就会出现乱码
        # unicode转换方法:  basename.encode('gb2312')  
        # utf-8转换方法:    basename.decode('utf-8').encode('gb2312')  
        filename = os.path.basename(f)
        part.add_header('Content-Disposition', 'attachment; filename="%s"' % filename.decode('utf8').encode('gb2312'))
        msg.attach(part)

    s.mail(mail_from)
    # now, rcpt_to is useless due to X_SMTPAPI
    s.rcpt(x_smtpapi['to'][0])
    reply_data = s.data(msg.as_string())
    s.quit()

    message_id = _message_id(reply_data)

    for rcpt_to in x_smtpapi['to']:
        ret[rcpt_to] = message_id + str(x_smtpapi['to'].index(rcpt_to)) + '$' + rcpt_to

    print ret
    return ret

def test_send():
    mail_from = 'anything@ifaxin.com' # useless
    ffrom = formataddr((str(Header(u'爱发信客服支持', 'utf-8')), "support@ifaxin.com"))
    rcpt_tos = ["ben@ifaxin.com", "joe@ifaxin.com"]
    reply_to = 'service@ifaxin.com'
    subject = '关于问题1901的回复'
    content = "请登录爱发信, 查看此问题的回复.  <br/> <a href='http://www.ifaxin.com'>http://www.ifaxin.com</a>"
    files = ['/path/file1', '/path/files2', ]

    send(mail_from, ffrom, rcpt_tos, reply_to, subject, content, files)

def test_sendn():
    mail_from = 'anything@ifaxin.com' # useless
    ffrom = formataddr((str(Header(u'爱发信客服支持', 'utf-8')), "support@ifaxin.com"))
    x_smtpapi = {
        "to": ["ben@ifaxin.com", "joe@ifaxin.com",],
        "sub": {
            "%name%": ["Ben", "Joe"],
            "%money%":[288, 497]
        },
        "filters": {
            "subscription_tracking": { "settings": { "enable": "1" } },
            "open_tracking"        : { "settings": { "enable": "1" } },
            "click_tracking"       : { "settings": { "enable": "1" } }
        }
    }
    reply_to = 'service@ifaxin.com'
    subject = "爱发信2015年1月份消费账单"
    content = "亲爱的%name%: <br/> 您好! 您本月在爱发信的消费金额为: %money% 元. <br/> <br/> <a href='http://www.ifaxin.com'>http://www.ifaxin.com</a>"
    files = ['/path/file1', '/path/files2', ]

    sendn(mail_from, ffrom, x_smtpapi, reply_to, subject, content, files)

def main():
    # send email one by one
    test_send()

    # send email with x_smtpapi
    test_sendn()

if __name__ == '__main__':                                                      
    main()
    
```
[downloads](../downloads/code/python_smtp.py)
    
