# 入侵检测-Snort

## 实验环境

1. kali-victim`172.16.111.107`

  ![attacker](img/attacker.png)

2. kali-attacker`172.16.111.130`

  ![victim](img/victim.png)

3. Gateway`172.16.111.1`

  ![gateway](img/Gateway.png)

## 实验过程

### 安装Snort

```bash
# 禁止在apt安装时弹出交互式配置界面
export DEBIAN_FRONTEND=noninteractive
apt install snort
```

![install](img/install.png)

### 实验一：配置snort为嗅探模式

```bash
#受害者Ping攻击者主机，攻击者主机结果如下
# 使用 CTRL-C 退出嗅探模式
# 显示IP/TCP/UDP/ICMP头
snort –v
```

![-v](img/v.png)

```bash
# 显示应用层数据
snort -vd
```

![-vd](img/vd.png)

```bash
# 显示数据链路层报文头
snort -vde
```

![-vde](img/vde.png)

```bash
# -b 参数表示报文存储格式为 tcpdump 格式文件
# -q 静默操作，不显示版本欢迎信息和初始化信息
snort -q -v -b -i eth0 "port not 22"
# 嗅探到的数据包会保存在 /var/log/snort/snort.log.<epoch timestamp>
# 其中<epoch timestamp>为抓包开始时间的UNIX Epoch Time格式串
date -d @<epoch timestamp> #转换时间为人类可读格式
# 上述命令用tshark等价实现如下：
tshark -i eth0 -f "port not 22" -w 1_tshark.pcap
```

![嗅探数据包](img/查看嗅探到的数据包.png)

### 实验二：配置并启用snort内置规则

```bash
# /etc/snort/snort.conf 中的 HOME_NET 和 EXTERNAL_NET 需要正确定义
# 例如，学习实验目的，可以将上述两个变量值均设置为 any
snort -q -A console -b -i eth0 -c /etc/snort/snort.conf -l /var/log/snort/
```

![any](img/any.png)

### 实验三：自定义snort规则

```bash
# 新建自定义 snort 规则文件
cat << EOF > /etc/snort/rules/cnss.rules
alert tcp \$EXTERNAL_NET any -> \$HTTP_SERVERS 80 (msg:"Access Violation has been detected on /etc/passwd ";flags: A+; content:"/etc/passwd"; nocase;sid:1000001; rev:1;)
alert tcp \$EXTERNAL_NET any -> \$HTTP_SERVERS 80 (msg:"Possible too many connections toward my http server"; threshold:type threshold, track by_src, count 100, seconds 2; classtype:attempted-dos; sid:1000002; rev:1;)
EOF
# 添加配置代码到 /etc/snort/snort.conf
include $RULE_PATH/cnss.rules
# 应用规则开启嗅探
snort -q -A fast -b -i eth1 -c /etc/snort/snort.conf -l /var/log/snort/
# 在attacker上使用ab命令进行压力测试
ab -c 100 -n 10000 http://172.16.111.107/hello
```
![嗅探](img/catch.png)

#### 实验四：和防火墙联动

```bash
# 解压缩 Guardian-1.7.tar.gz
tar zxf guardian.tar.gz

# 安装 Guardian 的依赖 lib
apt install libperl4-corelibs-perl
```

在Victim上先后开启 snort 和 guardian.pl
```bash
# 开启 snort
snort -q -A fast -b -i eth1 -c /etc/snort/snort.conf -l /var/log/snort/
# guardian.tar.gz 解压缩后文件均在 hoem/kali/workspace 下
cd /home/kali/workspace/guardian

# 编辑 guardian.conf 并保存，更改参数为
HostIpAddr      172.16.111.107
Interface       eth0
```
```bash
# 启动 guardian.pl
perl guardian.pl -c guardian.conf
```

在Attacker上用 `nmap` 扫描 Victim：

```bash
nmap 192.168.56.107 -A -T4 -n -vv
```

![guardian](img/guardian.png)

### 实验中遇到的问题

1. 实验三添加配置代码的时候我以为是直接在命令行里输入,include $RULE_PATH命令，结果因为include command not found卡了很久。但经过查看config文件后，发现是直接在配置文件中手动插入....
   
2. guardian.conf 中默认的来源IP被屏蔽时间是 60 秒（屏蔽期间如果黑名单上的来源IP再次触发snort报警消息，则屏蔽时间会继续累加60秒）

## 思考题解答：

1. IDS与防火墙的联动防御方式相比IPS方式防御存在哪些缺陷？是否存在相比较而言的优势？

答：IDS通过软、硬件，对网络、系统的运行状况进行监视，尽可能发现各种攻击企图、攻击行为或者攻击结果，以保证网络系统资源的机密性、完整性和可用性。
IPS是一部能够监视网络或网络设备的网络资料传输行为的计算机网络安全设备，能够即时的中断、调整或隔离一些不正常或是具有伤害性的网络资料传输行为。
也就是说，IDS偏向于检测，属于管理系统，会检测出不安全的网络行为，但是不阻断任何网络行为。IPS偏向于防御，属于控制系统，可以阻止危险行为。
举个形象的例子，防火墙可以比作一栋楼的门锁，IDS就是警告系统，可以在有人撬锁后嗅探到，IPS则会在嗅探到有人撬锁后把门冻结。

## 参考链接

[snort的安装、配置和使用](https://blog.csdn.net/qq_37865996/article/details/85088090)
[实验九在线课本](https://c4pr1c3.github.io/cuc-ns/chap0x09/exp.html)

