# tcpdump命令参数 <!-- {docsify-ignore-all} -->


## tcpdump命令格式几参数介绍

tcpdump 是一个运行在命令行下的抓包工具。它允许用户拦截和显示发送或收到过网络连接到该计算机的TCP/IP和其他数据包。tcpdump 适用于

大多数的类Unix系统操作系统(如linux,BSD等)。类Unix系统的 tcpdump 需要使用libpcap这个捕捉数据的库就像 windows下的WinPcap。

在学习tcpdump前最好对基本网络的网络知识有一定的认识。


tcpdump命令的格式为：

```powershell
Usage: tcpdump [-aAbdDefhHIJKlLnNOpqStuUvxX#] [ -B size ] [ -c count ]
		[ -C file_size ] [ -E algo:secret ] [ -F file ] [ -G seconds ]
		[ -i interface ] [ -j tstamptype ] [ -M secret ] [ --number ]
		[ -Q in|out|inout ]
		[ -r file ] [ -s snaplen ] [ --time-stamp-precision precision ]
		[ --immediate-mode ] [ -T type ] [ --version ] [ -V file ]
		[ -w file ] [ -W filecount ] [ -y datalinktype ] [ -z postrotate-command ]
		[ -g ] [ -k (flags) ] [ -o ] [ -P ] [ -Q meta-data-expression ]
		[ --apple-tzo offset ] [--apple-truncate ]
		[ -Z user ] [ expression ]
```

tcpdump的选项介绍：

```
　   -a 　　　将网络地址和广播地址转变成名字；
　　　-d 　　　将匹配信息包的代码以人们能够理解的汇编格式给出；
　　　-dd 　　　将匹配信息包的代码以c语言程序段的格式给出；
　　　-ddd 　　　将匹配信息包的代码以十进制的形式给出；
　　　-e 　　　在输出行打印出数据链路层的头部信息，包括源mac和目的mac，以及网络层的协议；
　　　-f 　　　将外部的Internet地址以数字的形式打印出来；
　　　-l 　　　使标准输出变为缓冲行形式；
　　　-n 　　　指定将每个监听到数据包中的域名转换成IP地址后显示，不把网络地址转换成名字；
     -nn：    指定将每个监听到的数据包中的域名转换成IP、端口从应用名称转换成端口号后显示
　　　-t 　　　在输出的每一行不打印时间戳；
　　　-v 　　　输出一个稍微详细的信息，例如在ip包中可以包括ttl和服务类型的信息；
　　　-vv 　　　输出详细的报文信息；
　　　-c 　　　在收到指定的包的数目后，tcpdump就会停止；
　　　-F 　　　从指定的文件中读取表达式,忽略其它的表达式；
　　　-i 　　　指定监听的网络接口；
     -p：    将网卡设置为非混杂模式，不能与host或broadcast一起使用
　　　-r 　　　从指定的文件中读取包(这些包一般通过-w选项产生)；
　　　-w 　　　直接将包写入文件中，并不分析和打印出来；
     -s snaplen         snaplen表示从一个包中截取的字节数。0表示包不截断，抓完整的数据包。默认的话 tcpdump 只显示部分数据包,默认68字节。
　　　-T 　　　将监听到的包直接解释为指定的类型的报文，常见的类型有rpc （远程过程调用）和snmp（简单网络管理协议；）
     -X            告诉tcpdump命令，需要把协议头和包内容都原原本本的显示出来（tcpdump会以16进制和ASCII的形式显示），这在进行协议分析时是绝对的利器。
```

## tcpdump命令的5个常用参数

❶-i参数。使用-i参数指定需要抓包的网卡。如果未指定的话，tcpdump会根据搜索到的系统中状态为UP的最小数字的网卡确定，一般情况下是eth0。使用-i参数通过指定需要抓包的网卡，可以有效的减少抓取到的数据包的数量，增加抓包的针对性，便于后续的分析工作。
 
❷-nnn参数。使用-nnn参数禁用tcpdump展示时把IP、端口等转换为域名、端口对应的知名服务名称。这样看起来更加清晰。
 
❸-s参数。使用-s参数，指定抓包的包大小。使用-s 0指定数据包大小为262144字节，可以使得抓到的数据包不被截断，完整反映数据包的内容。
 
❹-c参数。使用-c参数，指定抓包的数量。
 
❺-w参数。使用-w参数指定抓包文件保存到文件，以便后续使用Wireshark等工具进行分析。


## tcpdump过滤器

❶host a.b.c.d：指定仅抓取本机和某主机a.b.c.d的数据通信。
 
❷tcp port x：指定仅抓取TCP协议目的端口或者源端口为x的数据通信。
 
❸icmp：指定仅抓取ICMP协议的数据通信。
 
❹!：反向匹配，例如port ! 22，抓取非22端口的数据通信。 以上几种过滤器规则，可以使用and或者or进行组合，例如： host a.b.c.d and tcp port x：则只抓取本机和某主机a.b.c.d之间基于TCP的目的端口或者源端口为x的数据通信。 tcp port x or icmp：则抓取TCP协议目的端口或者源端口为x的数据通信或者ICMP协议的数据通信。
 
## 示例

```powershell
1、抓取包含10.10.10.122的数据包
# tcpdump -i eth0 -vnn host 10.10.10.122
  
2、抓取包含10.10.10.0/24网段的数据包
# tcpdump -i eth0 -vnn net 10.10.10.0/24
  
3、抓取包含端口22的数据包
# tcpdump -i eth0 -vnn port 22
  
4、抓取udp协议的数据包
# tcpdump -i eth0 -vnn  udp
  
5、抓取icmp协议的数据包
# tcpdump -i eth0 -vnn icmp
6、抓取arp协议的数据包
# tcpdump -i eth0 -vnn arp
  
7、抓取ip协议的数据包
# tcpdump -i eth0 -vnn ip
  
8、抓取源ip是10.10.10.122数据包。
# tcpdump -i eth0 -vnn src host 10.10.10.122
  
9、抓取目的ip是10.10.10.122数据包
# tcpdump -i eth0 -vnn dst host 10.10.10.122
  
10、抓取源端口是22的数据包
# tcpdump -i eth0 -vnn src port 22
  
11、抓取源ip是10.10.10.253且目的ip是22的数据包
# tcpdump -i eth0 -vnn src host 10.10.10.253 and dst port 22
                  
12、抓取源ip是10.10.10.122或者包含端口是22的数据包
# tcpdump -i eth0 -vnn src host 10.10.10.122 or port 22
  
13、抓取源ip是10.10.10.122且端口不是22的数据包
[root@ ftp]# tcpdump -i eth0 -vnn src host 10.10.10.122 and not port 22
 
14、抓取源ip是10.10.10.2且目的端口是22，或源ip是10.10.10.65且目的端口是80的数据包。
# tcpdump -i eth0 -vnn \( src host 10.10.10.2 and dst port 22 \) or   \( src host 10.10.10.65 and dst port 80 \)
  
15、抓取源ip是10.10.10.59且目的端口是22，或源ip是10.10.10.68且目的端口是80的数据包。
[root@localhost ~]# tcpdump -i  eth0 -vnn 'src host 10.10.10.59 and dst port 22' or  ' src host 10.10.10.68 and dst port 80 '
  
16、把抓取的数据包记录存到/tmp/fill文件中，当抓取100个数据包后就退出程序。
# tcpdump –i eth0 -vnn -w  /tmp/fil1 -c 100
  
17、从/tmp/fill记录中读取tcp协议的数据包
# tcpdump –i eth0 -vnn -r  /tmp/fil1 tcp
  
18、从/tmp/fill记录中读取包含10.10.10.58的数据包
# tcpdump –i eth0 -vnn -r  /tmp/fil1 host  10.10.10.58
```
