---
title: lorawan协议（中文版）
tags: lorawan
abbrlink: 61d6a201
date: 2018-12-05 21:34:09
---

# 介绍

网关和服务器之间的协议是有目的的非常基本的，仅用于演示目的，或用于私有和可靠的网络。

这里没有网关或服务器的认证，并且确认仅用于网络质量评估，而不是 纠正UDP数据报丢失（无重试）。

# 系统原理和相关定义

```
 ((( Y )))
     |
     |
+ - -|- - - - - - - - - - - - - +        xxxxxxxxxxxx          +--------+
| +--+-----------+     +------+ |       xx x  x     xxx        |        |
| |              |     |      | |      xx  Internet  xx        |        |
| | Concentrator |<--->| Host |<-------xx     or    xx-------->|        |
| |              | SPI |      | |      xx  Intranet  xx        | Server |
| +--------------+     +------+ |       xxxx   x   xxxx        |        |
|    ^                     ^    |           xxxxxxxx           |        |
|    | PPS +-------+ NMEA  |    |                              |        |
|    +-----|  GPS  |-------+    |                              +--------+
|          | (opt) |            |
|          +-------+            |
|                               |
|             Gateway           |
+- - - - - - - - - - - - - - - -+
```

- **网关**：无线电RX / TX板，基于Semtech多通道调制解调器（SX130x），收发器（SX135x）和/或低功耗独立调制解调器（SX127x）。
- **主机**：运行包转发器的嵌入式计算机。通过SPI链路驱动集中器。 GPS：具有“每秒1脉冲”的GNSS（GPS，伽利略，GLONASS等）接收器 输出和到主机的串行链接，以发送包含时间和地理坐标数据的NMEA帧。可选的。
- **网关**：由至少一个无线电集中器，主机，一些组成的设备网络连接到互联网或专用网络（以太网，3G，Wifi，微波链路），以及可选的GPS接收器进行同步。
- **服务器**：一种抽象计算机，它将处理由网关接收和转发的RF数据包，并发出RF数据包以响应网关必须发出的数据包。

假设网关可以在NAT后面或防火墙停止任何传入连接。 假设服务器具有静态IP地址（或通过DNS服务可解决的地址），并且能够接收特定端口上的传入连接。

# 上行协议

3.1 时序图

```
+---------+                                                    +---------+
| Gateway |                                                    | Server  |
+---------+                                                    +---------+
     | -----------------------------------\                         |
     |-| When 1-N RF packets are received |                         |
     | ------------------------------------                         |
     |                                                              |
     | PUSH_DATA (token X, GW MAC, JSON payload)                    |
     |------------------------------------------------------------->|
     |                                                              |
     |                                           PUSH_ACK (token X) |
     |<-------------------------------------------------------------|
     |                              ------------------------------\ |
     |                              | process packets *after* ack |-|
     |                              ------------------------------- |
     |                                                              |
```

## `PUSH_DATA` 包

网关使用该数据包类型主要是将所接收的RF分组和相关联的元数据转发到服务器。

| 字节    | 功能                    |
| ------- | ----------------------- |
| 0       | 协议版本2               |
| 1-2     | 随机凭证                |
| 3       | PUSH_DATA标识`0x00`     |
| 4-11    | 网关唯一标识（MAC地址） |
| 12-结束 | `JSON`对象，看第4章     |

## `PUSH_ACK`包

服务器使用该数据包类型立即确认收到的所有PUSH_DATA数据包。

| 字节 | 功能                                  |
| ---- | ------------------------------------- |
| 0    | 协议版本2                             |
| 1-2  | 与`PUSH_DATA`包中相同的凭证，用于确认 |
| 3    | `PUSH_ACK`标识`0x01`                  |

# 上行`JSON`数据结构

根对象包含名为`"rxpk"`的数组：

```
{
	"rxpk":[ {...}, ...]
}
```

该数组包含至少一个`JSON`对象，每个对象包含一个RF数据包以及包含以下字段的关联元数据：

| 名称 | 类别   | 功能                                                        |
| ---- | ------ | ----------------------------------------------------------- |
| time | string | UTC time of pkt RX, us precision, ISO 8601 ‘compact’ format |
| tmst | number | Internal timestamp of “RX finished” event (32b unsigned)    |
| freq | number | RX central frequency in MHz (unsigned float, Hz precision)  |
| chan | number | Concentrator “IF” channel used for RX (unsigned integer)    |
| rfch | number | Concentrator “RF chain” used for RX (unsigned integer)      |
| stat | number | CRC status: 1 = OK, -1 = fail, 0 = no CRC                   |
| modu | string | Modulation identifier “LORA” or “FSK”                       |
| datr | string | LoRa datarate identifier (eg. SF12BW500)                    |
| datr | number | FSK datarate (unsigned, in bits per second)                 |
| codr | string | LoRa ECC coding rate identifier                             |
| rssi | number | RSSI in dBm (signed integer, 1 dB precision)                |
| lsnr | number | Lora SNR ratio in dB (signed float, 0.1 dB precision)       |
| size | number | RF packet payload size in bytes (unsigned integer)          |
| data | string | Base64 encoded RF packet payload, padded                    |

示例（为了便于阅读而添加了空格，缩进和换行符）：

```
{"rxpk":[
	{
		"time":"2013-03-31T16:21:17.528002Z",
		"tmst":3512348611,
		"chan":2,
		"rfch":0,
		"freq":866.349812,
		"stat":1,
		"modu":"LORA",
		"datr":"SF7BW125",
		"codr":"4/6",
		"rssi":-35,
		"lsnr":5.1,
		"size":32,
		"data":"-DS4CGaDCdG+48eJNM3Vai-zDpsR71Pn9CPA9uCON84"
	},{
		"time":"2013-03-31T16:21:17.530974Z",
		"tmst":3512348514,
		"chan":9,
		"rfch":1,
		"freq":869.1,
		"stat":1,
		"modu":"FSK",
		"datr":50000,
		"rssi":-75,
		"size":16,
		"data":"VEVTVF9QQUNLRVRfMTIzNA=="
	},{
		"time":"2013-03-31T16:21:17.532038Z",
		"tmst":3316387610,
		"chan":0,
		"rfch":0,
		"freq":863.00981,
		"stat":1,
		"modu":"LORA",
		"datr":"SF10BW125",
		"codr":"4/7",
		"rssi":-38,
		"lsnr":5.5,
		"size":32,
		"data":"ysgRl452xNLep9S1NTIg2lomKDxUgn3DJ7DE+b00Ass"
	}
]}
```

根对象还可以包含名为`"stat"`的对象：

```
{
	"rxpk":[ {...}, ...],
	"stat":{...}
}
```

数据包可能不包含`"rxpk"`数组而是“stat”对象。

```
{
	"stat":{...}
}
```

该对象包含网关的状态，包含以下字段：

| 名称 | 类型   | 功能                                                         |
| ---- | ------ | ------------------------------------------------------------ |
| time | string | UTC ‘system’ time of the gateway, ISO 8601 ‘expanded’ format |
| lati | number | GPS latitude of the gateway in degree (float, N is +)        |
| long | number | GPS latitude of the gateway in degree (float, E is +)        |
| alti | number | GPS altitude of the gateway in meter RX (integer)            |
| rxnb | number | Number of radio packets received (unsigned integer)          |
| rxok | number | Number of radio packets received with a valid PHY CRC        |
| rxfw | number | Number of radio packets forwarded (unsigned integer)         |
| ackr | number | Percentage of upstream datagrams that were acknowledged      |
| dwnb | number | Number of downlink datagrams received (unsigned integer)     |
| txnb | number | Number of packets emitted (unsigned integer)                 |

示例（为了便于阅读而添加了空格，缩进和换行符）：

```
{"stat":{
	"time":"2014-01-12 08:59:28 GMT",
	"lati":46.24000,
	"long":3.25230,
	"alti":145,
	"rxnb":2,
	"rxok":2,
	"rxfw":2,
	"ackr":100.0,
	"dwnb":2,
	"txnb":2
}}
```

# 下行协议

## 时序图

```
+---------+                                                    +---------+
| Gateway |                                                    | Server  |
+---------+                                                    +---------+
     | -----------------------------------\                         |
     |-| Every N seconds (keepalive time) |                         |
     | ------------------------------------                         |
     |                                                              |
     | PULL_DATA (token Y, MAC@)                                    |
     |------------------------------------------------------------->|
     |                                                              |
     |                                           PULL_ACK (token Y) |
     |<-------------------------------------------------------------|
     |                                                              |

+---------+                                                    +---------+
| Gateway |                                                    | Server  |
+---------+                                                    +---------+
     |      ------------------------------------------------------\ |
     |      | Anytime after first PULL_DATA for each packet to TX |-|
     |      ------------------------------------------------------- |
     |                                                              |
     |                            PULL_RESP (token Z, JSON payload) |
     |<-------------------------------------------------------------|
     |                                                              |
     | TX_ACK (token Z, JSON payload)                               |
     |------------------------------------------------------------->|
```

## PULL_DATA包

网关使用该数据包类型来轮询来自服务器的数据。

此数据交换由网关初始化，因为如果网关位于NAT后面，服务器可能无法将数据包发送到网关。 当网关初始化交换机时，将打开通向服务器的网络路由，并允许数据包在两个方向上流动。 网关必须定期发送PULL_DATA数据包，以确保网络路由保持打开状态，以便服务器随时使用。

| Bytes | Function                                |
| ----- | --------------------------------------- |
| 0     | protocol version = 2                    |
| 1-2   | random token                            |
| 3     | PULL_DATA identifier 0x02               |
| 4-11  | Gateway unique identifier (MAC address) |

### `PULL_ACK` 包

服务器使用该数据包类型来确认网络路由是否已打开，以及服务器是否可以随时发送PULL_RESP数据包。

| Bytes | Function                                          |
| ----- | ------------------------------------------------- |
| 0     | protocol version = 2                              |
| 1-2   | same token as the PULL_DATA packet to acknowledge |
| 3     | `PULL_ACK` identifier `0x04`                      |

### PULL_RESP 包

服务器使用该数据包类型来发送必须由网关发出的RF数据包和相关元数据。

| Bytes | Function                                                   |
| ----- | ---------------------------------------------------------- |
| 0     | protocol version = 2                                       |
| 1-2   | random token                                               |
| 3     | PULL_RESP identifier 0x03                                  |
| 4-end | JSON object, starting with {, ending with }, see section 6 |

### TX_ACK 包

网关使用该分组类型向服务器发送反馈，以通知网关是否已接受或拒绝下行链路请求。 数据报可以选项包含一个JSON字符串，以提供有关acknoledge的更多详细信息。 如果没有JSON（空字符串），这意味着没有发生错误。

| Bytes  | Function                                                     |
| ------ | ------------------------------------------------------------ |
| 0      | protocol version = 2                                         |
| 1-2    | same token as the PULL_RESP packet to acknowledge            |
| 3      | TX_ACK identifier 0x05                                       |
| 4-11   | Gateway unique identifier (MAC address)                      |
| 12-end | [optional] JSON object, starting with {, ending with }, see section 6 |

## 下行`JSON`数据结构

------

PULL_RESP数据包的根对象必须包含名为“txpk”的对象：

```
{
	"txpk": {...}
}
```

该对象包含要发出的RF数据包以及与以下字段相关联的元数据：

| Name | Type   | Function                                                     |
| ---- | ------ | ------------------------------------------------------------ |
| imme | bool   | Send packet immediately (will ignore tmst & time)            |
| tmst | number | Send packet on a certain timestamp value (will ignore time)  |
| time | string | Send packet at a certain time (GPS synchronization required) |
| freq | number | TX central frequency in MHz (unsigned float, Hz precision)   |
| rfch | number | Concentrator “RF chain” used for TX (unsigned integer)       |
| powe | number | TX output power in dBm (unsigned integer, dBm precision)     |
| modu | string | Modulation identifier “LORA” or “FSK”                        |
| datr | string | LoRa datarate identifier (eg. SF12BW500)                     |
| datr | number | FSK datarate (unsigned, in bits per second)                  |
| codr | string | LoRa ECC coding rate identifier                              |
| fdev | number | FSK frequency deviation (unsigned integer, in Hz)            |
| ipol | bool   | Lora modulation polarization inversion                       |
| prea | number | RF preamble size (unsigned integer)                          |
| size | number | RF packet payload size in bytes (unsigned integer)           |
| data | string | Base64 encoded RF packet payload, padding optional           |
| ncrc | bool   | If true, disable the CRC of the physical layer (optional)    |

大多数字段都是可选的。如果省略字段，将使用默认参数。 示例（为便于阅读而添加了空格，缩进和换行符）：

```
{"txpk":{
	"imme":true,
	"freq":864.123456,
	"rfch":0,
	"powe":14,
	"modu":"LORA",
	"datr":"SF11BW125",
	"codr":"4/6",
	"ipol":false,
	"size":32,
	"data":"H3P3N2i9qc4yt7rK7ldqoeCVJGBybzPY5h1Dd7P7p8v"
}}
{"txpk":{
	"imme":true,
	"freq":861.3,
	"rfch":0,
	"powe":12,
	"modu":"FSK",
	"datr":50000,
	"fdev":3000,
	"size":32,
	"data":"H3P3N2i9qc4yt7rK7ldqoeCVJGBybzPY5h1Dd7P7p8v"
}}
```

TX_ACK数据包的根对象必须包含名为“txpk_ack”的对象：

```
{
	"txpk_ack": {...}
}
```

该对象包含有关相关PULL_RESP数据包的状态信息。

| Name  | Type   | Function                                                     |
| ----- | ------ | ------------------------------------------------------------ |
| error | string | Indication about success or type of failure that occured for downlink request. |

可能的错误有：

| Value            | Definition                                                   |
| ---------------- | ------------------------------------------------------------ |
| NONE             | Packet has been programmed for downlink                      |
| TOO_LATE         | Rejected because it was already too late to program this packet for downlink |
| TOO_EARLY        | Rejected because downlink packet timestamp is too much in advance |
| COLLISION_PACKET | Rejected because there was already a packet programmed in requested timeframe |
| COLLISION_BEACON | Rejected because there was already a beacon planned in requested timeframe |
| TX_FREQ          | Rejected because requested frequency is not supported by TX RF chain |
| TX_POWER         | Rejected because requested power is not supported by gateway |
| GPS_UNLOCKED     | Rejected because GPS is unlocked, so GPS timestamp cannot be used |

示例（为便于阅读而添加了空格，缩进和换行符）：

```
{"txpk_ack":{
	"error":"COLLISION_PACKET"
}}
```