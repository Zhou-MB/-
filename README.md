# 阿里云服务器被植入挖矿程序

## 16日早上被通知处理

邮件：

![image-20240417100223898](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417100223898.png)

端口22、3389、2222、22222阻断后没及时处理，自动开放后又阻断

![image-20240417100252207](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417100252207.png)

![image-20240417095227436](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417095227436.png)

短信相似通知

## 检查

该服务器长期闲置，在15日当天部署了一个`redis`服务并连接测试，当晚凌晨就出现异常了

![image-20240417095549842](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417095549842.png)

发现`/var/spool/mail/root`有可疑邮件，凌晨一点多开始重复出现，报告一个cron服务（定期执行的任务或命令）的结果

![image-20240417101151125](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417101151125.png)

`chatgpt`解析内容：该作业的目的是检查当前系统进程中是否存在包含字符串 `R7HS5lHRGQ` 的进程，如果不存在，则尝试连接到 `168.119.173.48` 的 `60142` 端口，并将连接的输入输出重定向到文件描述符6。从文件描述符6读取响应消息，并将其写入 `/tmp/R7HS5lHRGQ` 文件中。给该文件添加执行权限，然后用一大段看不懂的参数执行文件。

打开`R7HS5lHRGQ`文件，非常长且看不懂：

![image-20240417101446416](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417101446416.png)

top 命令查看进程，你小子...

![image-20240417104506533](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417104506533.png)

检查ssh，自己以前应该没有配置过ssh，`~/.ssh`目录下仅有authorized_keys文件，这时间说明了一切...

![image-20240417103900188](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417103900188.png)

打开发现一个不知道哪来的公钥，和可疑邮件里的用户类似

![image-20240417102036350](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417102036350.png)

## 处理

第一时间想删除公钥，发现文件无法编辑，提示该文件只读，尽管我是root

![image-20240417111332407](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417111332407.png)

![image-20240417103726848](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417103726848.png)

后面发现该文件添加了不可修改属性（immutable attribute），先修改属性，再删除

![image-20240417114410726](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417114410726.png)

`kill -9 2616357`杀死`R7HS5lHRGQ`这小子的进程，并删除可疑文件`R7HS5lHRGQ`：

![image-20240417105216981](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417105216981.png)

`ls -lt`查找最近生成的可疑文件，先看`/tmp`目录，发现一大堆从16日凌晨逐步生成的文件，查看为空文件。

google后发现`Jsvc`是用来运行tomcat的一个应用与库集合

![image-20240417110400084](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417110400084.png)

![image-20240417110448308](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417110448308.png)

十分钟左右，发现`R7HS5lHRGQ`文件又生成了！！！只得再删除：

![image-20240417110714035](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417110714035.png)

ssh公钥又有了...再删除![image-20240417110941247](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417110941247.png)

top查看，这小子果然又回来了

![image-20240417111732885](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417111732885.png)

再次kill，并观察top，发现kill之后jsvc，一些bash会启动

![image-20240417112249820](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417112249820.png)

联想到`jsvc`用于启动Tomcat，遂阿里云安全组关闭8080端口，之后的cpu使用率、带宽都下降了

![image-20240417114704914](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417114704914.png)

删除了`\tmp`里的所有文件，后来又生成了2个，这个`hsperfdata/2742048`里面也是看不懂很长的乱码

![image-20240417122443762](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417122443762.png)

做了这些操作`/var/spool/mail/root`又出现了相似的邮件

![image-20240417120520736](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417120520736.png)

不过这次尾巴上的提示是timed out状态

![image-20240417120449678](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417120449678.png)

找到定时作业`/etc/crontab/`,但创建时间是刚刚

![image-20240417140959422](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417140959422.png)

打开定时文件，每30分钟执行一次，由root用户执行，如果当前系统中没有包含字符串 "R7HS5lHRGQ" 的进程，打开一个TCP连接到 `168.119.173.48` 的 `60142` 端口，并将连接的输入输出重定向到文件描述符6。从文件描述符6读取响应消息，并将其写入 `/tmp/R7HS5lHRGQ` 文件中。给该文件添加执行权限，用一大段参数执行文件。

![image-20240417141122480](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417141122480.png)

尝试删除失败，改变不可修改属性也失败

![image-20240417141838739](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417141838739.png)

找到定时文件`/var/spool/cron/root`,创建时间也是刚刚，内容和上面一样，只是没有指定root用户，一样不能修改属性，删除不了

![image-20240417142453690](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417142453690.png)

没办法了，重启主机，重置root密码...总算给它两删掉了

![image-20240417151205662](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417151205662.png)

再删除`\tmp`下的所有文件和`~/.ssh/authorized_keys`文件，没办法，又有了

拒绝访问病毒文件里的IP

![image-20240417152349599](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20240417152349599.png)

`redis`服务修改更复杂的连接密码，重启

但愿这下能干净了...
