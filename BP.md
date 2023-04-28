# BP

# 基本概念

BP协议最基本的数据格式是束（Bundle）

束由主数据块（Primary Bundle Block）和载荷数据块(Payload Block)构成

- 主数据块中包含了生存时间、时间戳和目的节点ID等基本信息

<img width="236" alt="image" src="https://user-images.githubusercontent.com/98028423/235211385-25730417-64ac-4792-a1e9-f4f37ed0ee67.png">

1. 7表示bundle协议的bpv7版本，6表示bundle协议的bpv6版本
2. <img width="301" alt="image" src="https://user-images.githubusercontent.com/98028423/235211442-a8f42f77-5f23-4830-965d-4e81f1a07caf.png">
3. <img width="302" alt="image" src="https://user-images.githubusercontent.com/98028423/235211479-d902bd4f-90b6-4541-9d1e-fa9d0c6bc866.png">

- 载荷数据块包含了数据块长度、数据块控制标志位和有效载荷等信息。

BP协议包含了两种重要机制：存储转发和托管传输

- 托管传输是可选机制。存储转发主要是为了克服链路频繁中断的情况，当检测到链路发生中断或者未找到目的节点时，节点会将束暂存在本地缓存中而不是将其丢弃，当链路连接回复时将继续发送数据。
- 托管传输机制需要节点具有永久存储空间，可以提高数据在极限条件下成功交付的可能性。

<img width="423" alt="image" src="https://user-images.githubusercontent.com/98028423/235211534-e14f88fe-6534-4b19-9585-aa1b7826571b.png">



# ion实现

## 启动指令

- bpadmin 配置、启动、管理ion节点的bp协议，bpadmin进入命令模式、bpadmin xx.bp 执行.bp中的命令、bpadmin . 停止bp
- iondmin 配置、启动、管理和停止本地计算机上的ION节点
- ionsecadmin在本地计算机上配置和管理ion安全策略数据库
- ltpadmin ltp传输管理接口 同bpadmin，有ltpclock、ltpcounter client_ID、ltpdriver remoteEngineNbr clientId nbrOfCycles greenLength、ltpmeter remote_engine_nbr聚合和分割、

## RC文件

- 单个“#”开头的行表示注释以方便了解相应配置所起的作用
- 两个“##”开头的行即表示此行为注释号，同时也起到识别不同管理进程的作用，如ionadmin, ionsecadmin, ltpadmin, bpadmin, ipnadmin。

```bash
## begin ionadmin 
1 1 ionconfig
# 1代表是命令1 ，1代表是节点1，节点2记得修改

# start ion node
s

# Add a contact.
a contact +0 +86400 1 2 125000
# 1到2节点，0s开始，86400s结束，上行速率125000bytes/s
a contact +0 +86400 2 1 125000
# 2到1节点，0s开始，86400s结束，下行速率125000bytes/s

# Add a range.
a range +0 +86400 1 2 1
# 两个节点的物理距离，0s-86400s，连接1节点到2节点，并且假设距离时间为1光年内
# 双向的，定义一个就可以

m production 1000000
m consumption 1000000
m horizon +0 
# 消耗和生产的均值为1000000 bytes/second

## end ionadmin 

## begin ionsecadmin
1  
e 1  
## end ionsecadmin 


## begin ltpadmin 
1 10 4800000
# 命令1，会话最大进程为10，内存空间限制4800000byte

# Add a span. (a connection) 
a span 2 5 240000 5 240000 1400 240000 1 'udplso 10.0.0.2:1113' 
# 连接到2节点，5为最大会话进程，240000为最大输出块尺寸，后面5 240000同
# segment尺寸为1400byte，限制聚合层大小为240000，聚合时间为1s
# output，udp连接到10.0.0.2，端口1113

s 'udplsi 10.0.0.1:1113'
# input，本地节点10.0.0.1，监听1113
m screening n  
# 启用或禁用接收的LTP的筛选，n表示禁用，默认是禁用
w 1  
## end ltpadmin 

## begin bpadmin 
1 
a scheme ipn 'ipnfw' 'ipnadminep'

# Add endpoints.
a endpoint ipn:1.0 q
a endpoint ipn:1.1 q
a endpoint ipn:1.2 q
# 终端节点，工程名：节点号.服务号
# q表示对于每个接收到的bundle进行排队，而x表示立即丢弃这个bundle

# Add a protocol.
a protocol ltp 1400 100
# ltp协议，假设传输容量每帧1400字节，100字节开销

# Add an induct. (listen)
a induct ltp 1 ltpcli
# 接收ltp的数据，通过ltpcli实现的

# Add an outduct. (send to host2)
a outduct ltp 2 ltpclo
# ltp协议发送数据，通过ltpclo实现

# Start the daemons
s
# 开始bp协议
## end bpadmin 

## begin ipnadmin 
a plan 2 ltp/2
# 添加输出计划，bundle要传输到node number为2的节点，使用标识为“2”的ltp输出
## end ipnadmin

//UDPCL
//修改下列四处
a protocol udp 1400 100
a induct udp <Local IP Address>:<Local Port> udpcli
a outduct udp * udpclo
//UDP 协议是无连接协议,在这里无须指明目的 IP 地址,“ * ”表示任意地址
a plan <Number of Node> udp/*,<Destination IP Address>:<Destination Port>
//当 UDP 协议作汇聚层时,在增加发送计划的时候必须指明发送的目的 IP 地址和端口
//TCPCL UDPCL 端口4556
```

## Test
```bash
- bpcancel bp协议取消程序   bpcancel <source EID> <creation seconds> <creation count> <fragment offset> <fragment length>
- bpchat bp聊天测试程序 bpchat.c <source EID> <dest EID>发送文本
- bpclock 守护进程
- bpcounter 接收测试程序 bpcounter <own endpoint ID>
- bpdriver 传输测试程序 bpdriver <number of cycles> <own endpoint ID> <destination endpoint ID> [<payload size>] [t<Bundle TTL>]
- bpecho 接收测试程序 bpecho ownEndpointId 每相应接收，发回长度为2的“echo”数据单元
- bplist 列出排队的dundle
- bpsendfile bprecvfile 文件传输程序
- bpsource 传输测试程序bpsource<destination endpoint ID> ["<text>"]
```
