# 套接字编程作业5：ICMP Ping程序

## 作业描述

《计算机网络：自顶向下方法》中第四章末尾给出了此编程作业的简单描述：

> ping是一种流行的网络应用程序，用于测试位于远程的某个特定的主机是否开机和可达。它也经常用于测量客户主机和目标主机之间的时延。他的工作过程是：向目标主机发送ICMP“回显请求”分组（即ping分组），并且侦听ICMP“回显响应”应答（即pong分组）。ping测量RTT，记录分组丢失和计算多个ping-pong交换（往返时间的最小，平均，最大和标准差）的统计汇总。
>
> 在本实验中，你将用Python语言编写自己的ping应用程序。你的应用程序将使用ICMP。但为了保持程序的简单，你将不完全遵循RFC 1739中的官方规范。注意到你将仅需要写该程序的客户程序，因为服务器侧所需的功能构建在几乎所有的操作系统中。你能够在Web站点 http://www.awl.com/kurose-ross 找到本作业的全面细节，以及该Python代码的重要片段。

## 详细描述

**官方文档：[Socket5_ICMPpinger(chap4).pdf](Socket5_ICMPpinger(chap4).pdf)**

**翻译：[作业5-ICMPping程序-翻译.md](作业5-ICMPping程序-翻译.md)**

## 实现

使用Python中原生套接字，创建一个IP套接字，在代码中构造ICMP数据包，实现ping操作。

## 代码

**Ping.py**

```python
import socket
import os
import struct
import time
import select

ICMP_ECHO_REQUEST = 8


def checksum(str):
    csum = 0
    countTo = (len(str) / 2) * 2
    count = 0
    while count < countTo:
        thisVal = str[count + 1] * 256 + str[count]
        csum = csum + thisVal
        csum = csum & 0xffffffff
        count = count + 2
    if countTo < len(str):
        csum = csum + str[len(str) - 1].decode()
        csum = csum & 0xffffffff
    csum = (csum >> 16) + (csum & 0xffff)
    csum = csum + (csum >> 16)
    answer = ~csum
    answer = answer & 0xffff
    answer = answer >> 8 | (answer << 8 & 0xff00)
    return answer


def receiveOnePing(mySocket, ID, sequence, destAddr, timeout):
    timeLeft = timeout

    while 1:
        startedSelect = time.time()
        whatReady = select.select([mySocket], [], [], timeLeft)
        howLongInSelect = (time.time() - startedSelect)
        if whatReady[0] == []:  # Timeout
            return None

        timeReceived = time.time()
        recPacket, addr = mySocket.recvfrom(1024)

        # Fill in start
        header = recPacket[20: 28]
        type, code, checksum, packetID, sequence = struct.unpack("!bbHHh", header)
        if type == 0 and packetID == ID:  # type should be 0
            byte_in_double = struct.calcsize("!d")
            timeSent = struct.unpack("!d", recPacket[28: 28 + byte_in_double])[0]
            delay = timeReceived - timeSent
            ttl = ord(struct.unpack("!c", recPacket[8:9])[0].decode())
            return (delay, ttl, byte_in_double)
        # Fill in end

        timeLeft = timeLeft - howLongInSelect
        if timeLeft <= 0:
            return None


def sendOnePing(mySocket, ID, sequence, destAddr):
    # Header is type (8), code (8), checksum (16), id (16), sequence (16)

    myChecksum = 0
    # Make a dummy header with a 0 checksum.
    # struct -- Interpret strings as packed binary data
    header = struct.pack("!bbHHh", ICMP_ECHO_REQUEST, 0, myChecksum, ID, sequence)
    data = struct.pack("!d", time.time())
    # Calculate the checksum on the data and the dummy header.
    myChecksum = checksum(header + data)

    header = struct.pack("!bbHHh", ICMP_ECHO_REQUEST, 0, myChecksum, ID, sequence)
    packet = header + data

    mySocket.sendto(packet, (destAddr, 1))  # AF_INET address must be tuple, not str
    # Both LISTS and TUPLES consist of a number of objects
    # which can be referenced by their position number within the object


def doOnePing(destAddr, ID, sequence, timeout):
    icmp = socket.getprotobyname("icmp")

    # SOCK_RAW is a powerful socket type. For more details see: http://sock-raw.org/papers/sock_raw

    # Fill in start
    mySocket = socket.socket(socket.AF_INET, socket.SOCK_RAW, icmp)
    # Fill in end

    sendOnePing(mySocket, ID, sequence, destAddr)
    delay = receiveOnePing(mySocket, ID, sequence, destAddr, timeout)

    mySocket.close()
    return delay


def ping(host, timeout=1):
    # timeout=1 means: If one second goes by without a reply from the server,
    # the client assumes that either the client’s ping or the server’s pong is lost
    dest = socket.gethostbyname(host)
    print("Pinging " + dest + " using Python:")
    print("")
    # Send ping requests to a server separated by approximately one second

    myID = os.getpid() & 0xFFFF  # Return the current process i
    loss = 0
    for i in range(4):
        result = doOnePing(dest, myID, i, timeout)
        if not result:
            print("Request timed out.")
            loss += 1
        else:
            delay = int(result[0]*1000)
            ttl = result[1]
            bytes = result[2]
            print("Received from " + dest + ": byte(s)=" + str(bytes) + " delay=" + str(delay) + "ms TTL=" + str(ttl))
        time.sleep(1)  # one second
    print("Packet: sent = " + str(4) + " received = " + str(4-loss) + " lost = " + str(loss))

    return


ping("www.baidu.com")
```

**代码文件**

[Ping.py](source/Ping.py)

## 运行

运行环境为Python 3，而不是题目中示例代码所使用的Python 2。

运行后会在终端显示：

```shell
Pinging 14.215.177.38 using Python:

Received from 14.215.177.38: byte(s)=8 delay=15ms TTL=55
Received from 14.215.177.38: byte(s)=8 delay=15ms TTL=55
Received from 14.215.177.38: byte(s)=8 delay=15ms TTL=55
Received from 14.215.177.38: byte(s)=8 delay=15ms TTL=55
Packet: sent = 4 received = 4 lost = 0
```

