---
layout: post
title:  "iOS不越狱抓取蜂窝网络数据"
date:   2016-06-19 21:37:31 +0800
categories: jekyll update
---

### Remote Virtual Interface

在iOS 5以后增加了RVI(Remote Virtual Interface），它让我们使用OS X来抓取ios device上数据包。
基本的方法就是把设备通过USB连上mac上。然后为这台设备安装RVI，这个虚拟的在Mac上的网卡，就代表这台ios设备的使用网卡。然后在mac上跑抓包的工具，定位到这个虚拟的网卡上，来抓包。

(1)安装RVI，需要使用rvictl工具，以下步骤在mac的终端中操作：

	$ # First get the current list of interfaces.
	$ ifconfig -l
	lo0 gif0 stf0 en0 en1 p2p0 fw0 ppp0 utun0
	$ # Then run the tool with the UDID of the device.
	$ rvictl -s 74bd53c647548234ddcef0ee3abee616005051ed

	Starting device 74bd53c647548234ddcef0ee3abee616005051ed 	[SUCCEEDED]
	
	$ # Get the list of interfaces again, and you can see the new virtual
	$ # network interface, rvi0, added by the previous command.
	$ ifconfig -l
	lo0 gif0 stf0 en0 en1 p2p0 fw0 ppp0 utun0 rvi0

(2)安装成功后，此时其实可以用任何抓包工具来抓取。包括wireshark等。因为这时就会看到一个rvi0的网卡。不过今天我们介绍的是通过tcpdump来搞。
在终端中输入如下命令：

	sudo tcpdump -i rvi0 -n -s 0 -w dump.pcap tcp
解释一下上面重要参数的含义：  
-i rvi0 选择需要抓取的接口为rvi0（远程虚拟接口）  
-s 0 抓取全部数据包  
-w dump.pcap 设置保存的文件名称  
tcp 只抓取tcp包

当tcpdump运行之后，你可以在iOS设备上开始浏览你想抓取的App，期间产生的数据包均会保存到dump.pcap文件中，当想结束抓取时直接终止tcpdump即可。然后在mac中找到dump.pcap文件。用wireshark打开就ok。

(3)去掉RVI这个虚拟网卡，使用下面的命令：
	
	$ rvictl -x 74bd53c647548234ddcef0ee3abee616005051ed
	Stopping device 74bd53c647548234ddcef0ee3abee616005051ed [SUCCEEDED]

[参考连接](http://www.fanliugen.com/?p=351)
