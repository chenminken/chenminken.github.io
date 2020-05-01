---
title: python程序同步webdav网盘（坚果云、owncloud)
date: 2020-01-10 11:58:06
tags:
---

在编写一个Django项目的时候有定时备份数据文件的需求。因为项目运行在云服务器中，担心的是服务器挂掉，所以备份的地方不能是同一台服务器。在这个项目里自己去建一个服务器来管理备份数据显得没有必要。目前网络存储提供商有许多家，我选取的方案是编写python定时程序连接支持WebDAV协议的网盘，例如坚果云和owncloud。

分为两部分：

1. Python同步代码编写
2. Django定时任务编写

## Python同步代码编写

使用[webdavclient3](https://pypi.org/project/webdavclient3/)库来处理webDAV协议的部分。先安装：

```
pip install webdavclient3
```

然后在自己的项目的某个地方建立一个py文件。我选的是`[项目目录]\utils\backup\backup.py`

编写python代码：

```python
from webdav3.client import Client
from datetime import datetime
from webdav3.exceptions import LocalResourceNotFound

import math
# invoke this function every day.
def upload():
    options = {
        'webdav_hostname': "网盘地址，如果是坚果云，不能只输入/dav/路径，似乎这个文件夹不能访问，在下面再建一个文件夹，比如backup。网址中需要有backup的地址比如https://dav.jianguoyun.com/dav/backup",
        'webdav_login': "用户名",
        'webdav_password': "密码，如果是坚果云填写应用密码",
        'disable_check': True, #有的网盘不支持check功能
    }
    client = Client(options)
		# 我选择用时间戳为备份文件命名
    file_name = str(math.floor(datetime.now().timestamp())) + '.bak'
    try:
        # 写死的路径，第一个参数是网盘地址
        client.upload('backup/' + file_name, '本地地址，绝对路径')
        # 打印结果，之后会重定向到log
        print('upload at ' + file_name)
    except LocalResourceNotFound as exception:
        print('An error happen: LocalResourceNotFound ---'  + file_name)

# 如果是直接调用文件，执行upload()
if __name__ == '__main__':
    print('run upload')
    upload()
```

Python的代码相对简短。只需要在服务器的命令行执行`python upload.py`，备份文件自动上传到网盘。然而还不完整，这个程序的目的是定期执行。接下来结合Django的定期执行接口做到每日备份。



## Django定时任务编写

Django是目前很流行的python web服务框架。通过django自带的命令创建project和app（这一部分不讲）。

当项目建好后，使用`python manage.py`可以运行项目、数据库代码整合。开发者可以通过继承BaseCommand类来自定义命令，然后再通过django_crontab定期执行命令。或者不通过自定义命令，直接使用django_crontab定期执行函数。

### 创建backupCmd命令

在之前创建的app下新建management/commands目录，在该目录下新建`backupCmd.py`

```
from django.core.management.base import BaseCommand
from utils.backup.backup import upload

class Command(BaseCommand):
    def handle(self, *args, **options):
        upload()
```

当完成后，在项目根目录下执行`python manage.py backupCmd`就可以单次执行程序了。

在setting.py尾部添加:

```
# 运行定时函数，每天1点运行。
CRONJOBS = [
    ('0 01 * * *', 'utils.backup.backup','>> ~/test_crontab.log')
]
```
或
```
# 运行定时命令， 
CRONJOBS = [
    ('*/1 * * * *', 'django.core.management.call_command', ['backupCmd'], {}, '>> ~/test_crontab.log'),
]
```

然后执行`python manage.py crontab add`，定时任务加入其中。

当时间到达的时候，程序将自动运行。日志会输出到`~/test_crontab.log`中。

linux中的定时任务crontab的语法如下:

```
*  *  *  *  * command
分钟(0-59) 小时(0-23) 每个月的哪一天(1-31) 月份(1-12) 周几(0-6) shell脚本或者命令
```



对于使用到的网盘，希望能够付费支持一下。