## Chapter 12 利用Bro完成取证分析实验报告
------
### 安装bro
```apt-get install bro bro-aux```
### 实验环境基本信息
```
lsb_release -a
uname -a
bro -v
```
![](https://i.imgur.com/sbK6FYE.png)
### 使用bro完成对attack-trace.pcap数据包分析
- 编辑bro配置文件
	- 配置```/etc/bro/site/local.bro```，在文件尾部追加两行新配置代码
```
@load frameworks/files/extract-all-files
@load mytuning.bro
```
![](https://i.imgur.com/h3LHY04.png)<br>
![](https://i.imgur.com/qcL2Fkm.png)<br>
	- 在```/etc/bro/site/```目录下创建新文件mytuning.bro，内容为：
```
redef ignore_checksums = T;
#默认情况下，bro会丢弃没有合格校验和的数据包，所以如果想分析没有经过NIC计算的本地数据包，需要设置忽略校验码。
```
![](https://i.imgur.com/GNqs6FY.png)<br>
![](https://i.imgur.com/qSClt7L.png)
- 使用bro自动化分析pcap文件
```
bro -r attack-trace.pcap /etc/bro/site/local.bro
```
出现警告信息WARNING: No Site::local_nets have been defined. It's usually a good idea to define your local networks.
![](https://i.imgur.com/1JhEmbU.png)
解决上述警告信息，编辑mytuning.bro，增加一行变量定义即可<br>
![](https://i.imgur.com/VJG0h1J.png)<br>
![](https://i.imgur.com/jmjfpRQ.png)<br>
增加这行关于本地网络IP地址范围的定义对于本次实验来说会新增2个日志文件，会报告在当前流量（数据包文件）中发现了本地网络IP和该IP关联的已知服务信息。<br>
在attack-trace.pcap文件的当前目录下会生成一些.log文件和一个extract_files目录
![](https://i.imgur.com/xEWLKxX.png)
<br>extract_files目录下有一个文件<br>
![](https://i.imgur.com/0JxgzMU.png)
<br>将该文件上传到virustotal，发现匹配了一个历史扫描报告，且报告62个引擎报毒，可判定该文件是攻击者攻击受害者的文件，表明这是一个已知的后门程序
![](https://i.imgur.com/AVB5mnj.png)
- 通过阅读```/usr/share/bro/base/files/extract/main.bro```的源代码
```
if ( ! args?$extract_filename )
        args$extract_filename = cat("extract-", f$last_active, "-", f$source,
                                    "-", f$id);

    f$info$extracted = args$extract_filename;
```
我们了解到该文件名的最右一个-右侧对应的字符串FHUsSu3rWdP07eRE4l是files.log中的文件唯一标识。
![](https://i.imgur.com/L4J8oxh.png)
- 通过查看files.log，发现该文件提取自网络会话标识（bro根据IP五元组计算出的一个会话唯一性散列值）为FHUsSu3rWdP07eRE4l的FTP会话。
![](https://i.imgur.com/oszTMaP.png)
由输出结果可知，该文件的来源ip为98.114.205.102,该文件的接收方ip是192.150.11.111。该文件提取自网络会话标识 CpUlIf1jgwDnpm8Jyl的会话。
- 查看ftp.log文件
![](https://i.imgur.com/azRulIc.png)
<br>发现被攻击者主机192.150.11.111向攻击者主机98.114.205.102请求下载ssms.exe的可执行文件。
<br>该传输过程发生在网络会话标志为C29zAw3c0RR9SPYLt5的会话中,其中ftp会话的user和password均为1。
- 查看conn.log文件
![](https://i.imgur.com/n2ObrSj.png)
<br>其中发现了受害者主机向攻击者主机请求下载ssms.exe的会话C29zAw3c0RR9SPYLt5和有害文件传输的会话CpUlIf1jgwDnpm8Jyl
<br><br>攻击过程大致为 攻击者（98.114.205.102）尝试连接受害者（192.150.11.111），并进行漏洞利用，控制受害者电脑执行指令，使受害者向攻击者主动发送ftp下载请求，下载有害文件ssms.exe,随后攻击者主机将有害文件通过ftp传输给受害者
- 查看dpd.log文件
![](https://i.imgur.com/JOyJZiq.png)
<br>输出结果中检测到了SMB协议识别

-------
### 参考资料
- [使用bro来完成取证分析](http://sec.cuc.edu.cn/huangwei/textbook/ns/chap0x12/exp.html)
- [Frequently Asked Questions  ___ Why isn’t Bro producing the logs I expect? (a note about checksums)](https://www.bro.org/documentation/faq.html#why-isn-t-bro-producing-the-logs-i-expect-a-note-about-checksums)
-  [virustotal.com](https://www.virustotal.com/#/home/upload)
-  [chap0x12 实战Bro网络入侵取证.md](https://github.com/CUCCS/2018-NS-Public-jackcily/blob/dfffc18a33c62f768496c0d77b227ec8d1b1a135/chap0x12%20%E5%AE%9E%E6%88%98Bro%E7%BD%91%E7%BB%9C%E5%85%A5%E4%BE%B5%E5%8F%96%E8%AF%81.md)