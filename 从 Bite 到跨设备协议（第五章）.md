# 系统中的数据表达、编码与传输：从 Bite 到跨设备协议 五章

## 第 5 章 · 第 1 课时

**Frame 是什么？为什么设备通信一定需要“帧”**

这一章的主题是：

> **如何把一串连续的 Byte，组织成接收方能够可靠识别、解析、校验的数据包。**
>

它会把前 4 章真正串起来。

### 一、先回顾前 4 章解决了什么

前面我们已经知道：

```text
Bit
 ↓
Byte
 ↓
二进制数据
 ↓
整数 / 字符 / UTF-8 / Struct
 ↓
Raw Binary / Hex / Base64
```

但现在有一个新的问题。

假设 MCU 通过 UART 发：

```text
01 02 03 04
```

接收方只能看到：

```text
01 02 03 04
```

它不知道：

- `01` 是什么意思？
- `02` 是命令吗？
- `03 04` 是一个 uint16\_t 吗？
- 一条消息从哪里开始？
- 在哪里结束？
- 如果连续来了两条消息怎么办？
- 如果中间少了一个 Byte 怎么办？

所以：**Byte 本身只是数据。**

要让 Byte 具有通信意义，需要在 Byte 之上定义协议结构。

这个协议结构最常见的基本单位，就是：**帧**

### 二、Frame —— 帧

可以先把 Frame 理解成：

**一条具有明确边界和内部格式的完整协议消息。**

例如我们设计一个非常简单的协议：

```text
+--------+--------+---------+
| Header | Command| Payload |
+--------+--------+---------+
```

实际发送：

```text
AA 01 12 34
```

规定：

```text
AA       Header
01       Command
12 34    Payload
```

那么这一整组：

```text
AA 01 12 34
```

就是一帧。

### 三、Header：告诉接收方“这里是一帧的开始”

最简单的方法：

```text
AA 55
```

作为固定帧头。

例如：

```text
AA 55 10 01
```

含义：

```text
AA 55   Header
10      Command
01      Payload
```

于是接收方不断监听 Byte：

```text
39
12
87
AA
55
10
01
```

它看到：

```text
AA 55
```

就知道：**“可能有一帧开始了。”**

这里有一个非常重要的概念：**同步 Sync**

Header 经常也叫：

```text
Sync Word
Magic Number
Start Of Frame
SOF
Preamble
```

不同协议名字不同，本质类似：

帮助接收方在连续 Byte Stream 中找到帧起点。

### 四、但只有 Header 仍然不够

假设我们收到：

```text
AA 55 10 01 02 03 04 05 06 ...
```

知道：

```text
AA 55
```

是开始。

但是：**什么时候结束？**

所以协议通常还需要：**Length**

例如定义：

```text
+--------+--------+---------+---------+
| Header | Length | Command | Payload |
+--------+--------+---------+---------+
```

假设：

```text
AA 55 03 10 12 34
```

规定：

```text
AA 55    Header
03       Length
10       Command
12 34    Payload
```

这里假设 Length 表示：

```text
Command + Payload
```

所以，接收方看到：

```text
AA 55
```

找到帧头。

然后读：

```text
03
```

于是知道：**后面还有 3 Byte 属于这一帧。**

因此：

```text
AA 55 03 10 12 34
```

就是完整的一帧。

这已经开始有真正协议的样子了。

### 五、再加 CRC

通信过程可能出错，例如发送：

```text
AA 55 03 10 12 34
```

接收到：

```text
AA 55 03 10 12 35
```

最后一个 bit 发生了错误。

如果没有校验：**接收方可能完全不知道数据已经坏了。**

所以增加：

```text
CRC
```

结构变成：

```text
+--------+--------+---------+---------+------+
| Header | Length | Command | Payload | CRC  |
+--------+--------+---------+---------+------+
```

例如：

```text
AA 55 03 10 12 34 8F
```

现在这一帧里已经有了几个核心信息：

```text
AA 55
│
└── 我从哪里开始？

03
│
└── 这一帧有多长？

10
│
└── 你要我干什么？

12 34
│
└── 命令携带的数据是什么？

8F
│
└── 数据有没有传坏？
```

### 六、以后看到的大多数私有协议，本质都逃不开这些东西

例如：

```text
+--------+---------+---------+---------+------+------+
| Header | Version | Length  | Command | Data | CRC  |
+--------+---------+---------+---------+------+------+
```

或者：

```text
+------+-----+-----+------+---------+------+-----+
| SOF  | SEQ | CMD | LEN  | Payload | CRC  | EOF |
+------+-----+-----+------+---------+------+-----+
```

虽然名字不同，但核心问题始终只有几个：

```text
1. 从哪里开始？
2. 数据有多长？
3. 这是什么命令？
4. 命令的数据是什么？
5. 数据有没有损坏？
6. 多帧连续到达时怎么区分？
```

### 七、Frame 和 Struct 区别

这是一个非常重要的问题。

例如 C 中：

```c
typedef struct
{
    uint16_t header;
    uint8_t  len;
    uint8_t  cmd;
    uint16_t value;
} packet_t;
```

看起来很像 Frame。

但：

- **Struct 是内存结构。**
- **Frame 是通信协议结构。**

两者不是同一个概念。

因为 Struct 可能存在：

```text
Padding
Alignment
CPU Endianness
编译器差异
```

真正的协议通常应该明确规定：

```text
Byte 0      Header[0]
Byte 1      Header[1]
Byte 2      Length
Byte 3      Command
Byte 4..N   Payload
最后        CRC
```

也就是：**协议描述的是线上的 Byte 格式，而不是某个 C Struct 的内存布局。**

这个思想会贯穿后面的序列化章节。

### 八、建立一个非常重要的分层意识

以后看到通信问题，建议脑子里始终保持：

```text
应用语义
    ↓
Protocol
    ↓
Frame
    ↓
Byte
    ↓
Bit
    ↓
物理链路
```

比如：

```text
“打开 LED”    --->    **应用语义** # 抽象的业务逻辑/指令意图


# 转换
CMD = 0x10
VALUE = 0x01    --->  **Protocol(协议)** # 将业务逻辑转化为标准化、无歧义的数据结构与字段


# 组帧
AA 55 02 10 01 CRC    --->  **Frame** # 按协议规矩打包好、带有边界和校验的完整数据单元


# 由 UART 发
AA
55
02
10
01
CRC                --->  **Byte** # 拆分成串口/硬件能处理的连续字节流


# UART TX 引脚上传输
0 / 1 电平            --->  **Bit** # 物理引脚上的电压变化
```

---

## 第 5 章 · 第 2 课时

上一课我们建立了一个核心概念：

> **Frame 是一条具有明确边界和内部结构的完整协议消息。**
>

今天我们继续解决一个非常关键的问题：**一帧到底应该多长？**

最常见的设计方式有两种：

```text
1. 固定长度帧
2. 可变长度帧
```

这两种方案看起来只是“长度是否固定”的区别，但实际上会直接影响：

- 协议设计复杂度
- 接收端解析难度
- 带宽利用率
- 错误恢复能力
- MCU RAM 占用
- 后续扩展性

### 一、固定长度帧

假设我们设计一个非常简单的 MCU 协议。

每一帧永远固定为 8 Byte：

```text
+--------+-----+------+---------+--------+
| Header | CMD | Data | Reserve | CRC    |
+--------+-----+------+---------+--------+
   2B      1B    2B      2B       1B
```

总长度：8 Byte

例如：

```text
AA 55 10 12 34 00 00 8F
```

定义：

```text
AA 55    Header
10       CMD
12 34    Data
00 00    Reserve
8F       CRC
```

那么接收方知道：只要找到 `AA 55`，接下来凑够 8 Byte，就得到一整帧。

#### 固定长度帧优点一

好解析，假设 UART 收到：

```text
AA 55 10 01 00 00 00 8F
AA 55 20 64 00 00 00 92
AA 55 30 00 00 00 00 A7
```

每帧 8 Byte。

解析器只需要：

```text
找到 Header
    ↓
等待 8 Byte
    ↓
校验 CRC
    ↓
处理
```

逻辑非常简单。

甚至在某些非常简单的协议里：

```c
uint8_t frame[8];
```

接收满 8 Byte 后直接处理即可。

RAM 好规划，MCU 很喜欢这种确定性

#### 固定长度帧优点二

简单，尤其适合 MCU，例如：

```text
Frame Size = 16 Byte
```

接收端可以准备：`uint8_t rx_frame[16];`

然后：

```text
收 Byte
 ↓
计数
 ↓
16 Byte 收满
 ↓
处理
```

代码会非常容易写。

#### 固定帧长度优点三

没有复杂 Length 判断，接收端知道：

```text
帧起点 + 固定 N Byte = 完整 Frame
```

所以状态机非常简单。

#### 固定长度帧的问题

**浪费带宽**，假设我们的 Frame 固定为：

```text
32 Byte
```

现在有一个命令：

```text
RESET
```

实际上只需要：CMD = 0x01

Payload 一个 Byte 都不需要。

但是协议要求：

```text
32 Byte
```

那么可能变成：

```text
AA 55 01 00 00 00 00 00 00 00 ...

后面大量都是：Padding
```

这样效率就很低。

### 二、可变长度帧

现在我们设计：

```text
+--------+--------+-----+---------+------+
| Header | Length | CMD | Payload | CRC  |
+--------+--------+-----+---------+------+
```

其中：

```text
Payload 长度可以变化
```

例如 RESET 就可以变成：

```text
AA 55 01 01 CRC
```

每一帧长度不同。

#### 接收端的解析

这是以后 Frame Parser 的基础，假设接收端逐 Byte 收到：

```text
AA
55
00
05
20
11
22
33
44
8A
7C
```

解析过程大概是：

```text
收到 AA
 ↓
可能是 Header 第一个 Byte

收到 55
 ↓
确认 Header

读取 Length = 5
 ↓
知道后面有 5 Byte 的 Command + Payload

继续收够 5 Byte
 ↓
20 11 22 33 44

再读取 CRC
 ↓
8A 7C

校验 CRC
 ↓
完整 Frame
```

所以，Length 字段实际上是在告诉 Parser：

**“这帧后面还有多少内容需要等待。”**

#### 可变长度帧的代价

很多初学者会觉得：**那肯定所有协议都应该用可变长度啊。**

可变长度帧有一个重要代价：**Parser 更复杂**

固定长度：

```text
找到帧头
 ↓
收够 N Byte
 ↓
完成
```

可变长度：

```text
找到帧头
 ↓
读取 Length
 ↓
判断 Length 是否合法
 ↓
根据 Length 等待数据
 ↓
防止 Length 导致 buffer 越界
 ↓
收完整 Payload
 ↓
读 CRC
 ↓
校验
```

复杂度明显更高。

#### Length 字段可能成为攻击入口

这个问题以后做工业级 Parser 时非常重要，假设协议：

```text
Length = uint16_t
```

最大理论值：**65535**

攻击者或者噪声导致收到：

```text
AA 55 FF FF
```

Parser 解析：

```text
Length = 65535
```

如果你的代码直接：

```c
memcpy(payload, rx, len);

// 而 实际上只有 256 Byte
```

那就可能发生：**buffer overflow**

所以真正工业级代码一定会判断：

```c
if (len > MAX_PAYLOAD_SIZE)
{
    // invalid frame
}
```

也就是说：**可变长度协议虽然灵活，但一定要配合严格的长度检查。**

### 三、更适合固定长度帧

#### 1. 数据非常简单

例如传感器：

```text
温度 + 湿度 + 状态
```

每次都固定：

```text
8 Byte
```

#### 2. 实时性要求很高

因为固定长度：

```text
CPU 开销小
Parser 简单
时间确定
```

很适合一些控制系统。

#### 3. 通信带宽不是问题

例如：

```text
SPI
CAN 某些上层应用
片内通信
板内通信
```

如果每次就多几个 Byte，可能完全不值得为了节省带宽增加复杂度。

#### 4. 命令数量少

例如只支持：

```text
READ
WRITE
RESET
STATUS
```

那么固定长度可能反而更加可靠。

### 四、更适合可变长度帧

#### 1. Payload 差异非常大

例如：

```text
GET_STATUS     0 Byte
SET_SN         16 Byte
WRITE_CERT     1024 Byte
UPGRADE_DATA   4096 Byte
```

#### 2. 需要扩展性

以后命令可能不断增加。

#### 3. 带宽有限

比如：

```text
UART 115200
BLE
低速无线
RS485
```

少发一个 Byte 都可能有价值。

#### 4. 网络通信

例如：

```text
TCP
UDP 上层自定义协议
```

大多数时候 Payload 大小变化很明显。

### 五、可变长度不一定必须有 Length 字段

可变长度 Frame 必须有某种“结束判定机制”，但不一定非得叫 Length。

还可以使用：

```text
特殊结束符

# 例如：
    \n
```

文本协议经常这样做，例如：

```text
HELLO\n
```

所以接收端：

```text
不断收
 ↓
直到发现 '\n'
 ↓
认为一条消息结束
```

这也是可变长度协议。

### 六、协议选择

首先问三个问题：

```text
1. 不同命令的数据长度差异大不大？
2. 最大 Payload 有多大？
3. 以后协议会不会扩展？
```

例如：

```text
LED 控制
继电器控制
状态查询
```

每条命令都只有：

```text
1~4 Byte
```

那固定长度完全可能更合适。

但如果涉及：

```text
SN
证书
密钥
文件
升级包
日志
配置数据
```

长度差别非常大，那么：**几乎一定应该考虑可变长度 Frame。**

### 七、设计一个使用的教学协议

后面几课我们可以一直围绕这个协议展开，暂时定义：

```text
+--------+--------+---------+---------+------+
| Header | Length | Command | Payload | CRC  |
+--------+--------+---------+---------+------+
```

字节布局：

```text
Byte 0      0xAA
Byte 1      0x55

Byte 2~3    Length    # Length = Command + Payload
Byte 4      Command

Byte 5~N    Payload

最后 2 Byte CRC16
```

注意，这里有一个设计问题

我们刚才说：`Length = Command + Payload`

但是协议完全也可以规定：

```text
Length = Payload
```

甚至可以规定：

```text
Length = 整个 Frame
```

不同协议对 `Length` 的理解可能完全不同。

这会直接影响 Parser。

所以：**Length 字段怎么定义，是协议设计中非常关键的一件事。**

这个问题我们不会现在一带而过。

后面专门用一整课讲。

---

## 第 5 章 · 第 3 课时

Header / Sync Word / SOF：接收端怎样找到一帧的开始

上一课我们知道：

```text
Header + Length + Command + Payload + CRC
```

可以组成一帧。

但这里有一个非常现实的问题：

> 接收端凭什么知道当前收到的这个 Byte，就是 Frame 的第一个 Byte？
>

假设 UART 接收端启动时，线上已经有数据：

```text
36 91 27 AA 55 03 10 11 22 8F 7C
```

前面：

```text
36 91 27
```

是什么？

可能是：

- 上一帧残留
- 噪声
- 接收程序启动晚了
- 之前解析出错留下的数据
- 串口异常产生的数据
- 上一帧的一部分

所以必须解决：**帧同步**

### 一、同步

这里的同步不是时钟同步，而是：

> **接收端在连续 Byte Stream 中重新找到协议帧的正确边界。**
>

英文经常看到：

```text
Frame Synchronization
Framing
Resynchronization
```

例如：

```text
12 83 47 AA 55 03 10 11 22 CRC
         └──── Frame ────────┘
```

Parser 要不断寻找：

```text
AA 55
```

一旦找到，才认为：**“这里可能是一帧的开始。”**

注意这里说的是：可能

### 二、Header

```text
AA 55
```

的主要作用之一就是：**帮助接收端寻找帧起点。同步标识**

这种特殊标记经常有很多名字：

```c
Header
SOF            // Start Of Frame
Sync
Sync Word
Magic
Magic Number
Preamble
Start Code
```

不同协议定义并不完全相同，我们先建立基本认识。

Header 不一定等于整个协议头

有时候文档可能写：

```text
Header = AA 55
```

但另一个文档可能把：

```text
AA 55 00 03 10
```

整个都叫：Protocol Header

所以以后看协议文档，一定要看：

> **这个文档具体怎么定义 Header。**
>

Header 不能随便选

假设我们定义：

```text
Header = 00
```

这当然可以。

但是 Payload 中：

```text
00
```

太常见了。

那么 Parser 会非常频繁地误认为：

```text
发现 Header 了！
```

所以通常希望同步字（帧头）：**在随机数据中不容易自然出现。**

我们可以很直观的理解：Sync Word 越长，误同步概率通常越低。

但也不是越长越好，因为：

```text
Header 越长
→ 每帧额外开销越大
```

最常见的：**AA 55**

### 三、找到 Header 不能认定 Frame 正确

假设 Parser 看到了帧头：**AA 55**

也不能就确定找到 Frame，因为这两个 Byte 可能只是：

> Payload 或随机噪声中恰好出现的。
>

所以真正的 Parser 会继续验证：

```text
Header 正确
   ↓
Length 合法？
   ↓
Command 合法？
   ↓
数据长度完整？
   ↓
CRC 正确？
   ↓
才最终认为是有效 Frame
```

这是非常重要的思想：**Header 只是第一道筛选**

### 四、Resynchronization 重新同步。

假设真实 Byte Stream：

```text
78 91 AA 55 FF FF 10 28 37
```

Parser 找到：**AA 55**

然后读到：**FF FF**

如果协议规定：**MAX\_PAYLOAD = 1024**

那么立即知道：

```text
Length 不合法
```

所以：这个 **AA 55** 不是真的帧头

于是 Parser ：**放弃当前候选帧，继续寻找下一个 Header。**

这就叫：重新同步

工业级 Parser 必须能够：

```text
错误发生
 ↓
丢弃错误候选数据
 ↓
继续扫描
 ↓
重新找到 AA 55
 ↓
恢复正常解析
```

也就是说：**一个好的协议 Parser 不能要求“从上电开始一个 Byte 都不能错”。**

它必须能够：**自恢复（重新同步）**

### 五、简单的 Header 搜索算法

如果：

```text
HEADER = AA 55
```

```c
if (byte == 0xAA)
{
    // 等待下一个 byte
}
```

如果下一字节：

```c
byte == 0x55
```

那么：**Header 找到了**

否则重新开始

可以抽象成状态机：

```text
WAIT_AA
   │
   │ 收到 AA
   ▼
WAIT_55
   │
   │ 收到 55
   ▼
HEADER_FOUND
```

如果在 `WAIT_55` 时收到别的：

```text
WAIT_55
   │
   │ 非 55
   ▼
WAIT_AA
```

这就是最简单的：**Frame Parser State Machine**

已经说了这是最简单的，说明它肯定不足以应付复杂的情况

不过对于 **MCU** 私有协议来说，足够了。暂时就先这样理解（只需要知道还有更复杂的）

Header 出现在 Payload 里，也是正常的

因为 Parser 一旦已经确定：

```text
当前 Frame Length
```

它就应该知道：

```text
Payload 有多少 Byte
```

所以一个典型状态机：

```text
SEARCH_HEADER
     ↓
READ_LENGTH
     ↓
READ_BODY
     ↓
READ_CRC
```

当处于：

```text
READ_BODY
```

那么此时的 **AA 55** 就只是 *Payload*

Payload 中的 Header 会产生麻烦

主要是在：**当前帧已经损坏、丢 Byte、Length 错误等情况。**

例如：

```text
AA 55 20 10 11 22 AA 55 ...
```

Length 被错误解析为：

```text
0x20
```

Parser 就可能一直等待 32 Byte。

那么中间真正下一帧的：

```text
AA 55
```

就被当成 Payload 了。

所以工业级设计还会考虑：

- 接收超时
- 最大帧长度
- CRC
- Length 合法性
- 异常重新同步

以后会专门讲。

### 六、一个比较成熟的 Header 长什么样

我们后续可以逐渐把教学协议扩展成：

```text
+--------+---------+--------+------+-----+---------+------+
| Magic  | Version | Length | Seq  | CMD | Payload | CRC  |
+--------+---------+--------+------+-----+---------+------+
|  2 B   |   1 B   |  2 B   | 2 B  | 1 B |  N B    | 2 B  |
+--------+---------+--------+------+-----+---------+------+
```

例如：

```text
AA 55
01
00 04
00 12
10
11 22 33
XX XX
```

这里：

```text
AA 55        # 是：**Magic / Sync Word**
```

而：

```text
AA 55 01 00 04 00 12 10
```

可以整体称：**Fixed Header**

### 七、UART 和 TCP 尤其需要这个概念

> **UART** 和 **TCP** 都是“流式传输字节”的接口，应用层看到的就是一串连续的字节流。
>
> 没有天然的“消息边界”
>

以 TCP 为例，TCP 只给你：

```text
Byte Stream
```

可能第一次：

```text
recv()
```

得到：

```text
AA 55 03
```

第二次得到：

```text
10 11
```

第三次得到：

```text
22 CRC
```

甚至可能程序刚启动时第一次拿到：

```text
11 22 CRC AA 55 03
```

所以 Parser 必须独立维护：

```text
我现在在哪里？
是否已经同步？
正在等待什么？
```

这就是为什么后面会进入：**状态机**

一个基本 Parser 状态机，可以设计为：

```text
SEARCH_HEADER
      ↓
READ_LENGTH
      ↓
READ_BODY
      ↓
READ_CRC
      ↓
CHECK_FRAME
      ↓
DISPATCH
```

异常：

```text
CRC Error
Length Error
Timeout
```

都能：

```text
↓
SEARCH_HEADER
```

这就是一个：

> 可以持续工作、出错后自动恢复的流式 Parser。
>

### 八、这一课最关键的一句话

你现在要牢牢记住：

> **Header 的作用不是证明“这一帧一定正确”，**
>
> **而是告诉 Parser：“这里值得尝试按一帧来解析。”**
>

真正判断 Frame 是否有效，需要：

```text
Magic
+
Length
+
字段合法性
+
CRC
```

共同完成。这是从“能跑的协议”走向“可靠协议”非常重要的一步。

---

## 第 5 章 · 第 4 课时

Length 字段到底应该表示什么？

### 一、Length 字段

我们假设 Frame：

```text
+--------+--------+---------+---------+-------+
| Magic  | Length | Command | Payload | CRC16 |
+--------+--------+---------+---------+-------+
|  2 B   |  2 B   |   1 B   |  N B    |  2 B  |
+--------+--------+---------+---------+-------+
```

最常见的 Length 定义有三种。

**方案 1：Length = Payload 长度**

这种定义的好处是：

> Length 直接告诉你业务数据有多少 Byte。
>

**方案 2：Length = Command \+ Payload**

比较常用的一种

**方案 3：Length = 整个 Frame 长度**

这种情况下，Parser 一看 Length，就直接知道：

> 从当前 Frame 起点开始，总共应该收 多少 Byte。
>

所以当别人给你一个协议文档时：

> 绝对不能只看到 Length 字段就自己猜它的意思。
>

必须明确问：

```text
Length counts what?
```

同理我们写 协议文档 必须写得非常明确

不要只写：

```text
Length: 2 bytes
```

应该写成类似：

```text
Length:
2-byte unsigned integer, big-endian.
Represents the number of bytes from Command through the end of Payload.
CRC is not included.
```

现实世界里：

```text
Length = Payload
Length = CMD + Payload
Length = Length + CMD + Payload
Length = Entire Frame
Length = Everything after Length
```

五花八门。

### 二、需要：MAX_LENGTH 与 MIN_LENGTH

**MAX\_LENGTH** ：

1. 防止 Buffer Overflow

    假设：

    ```c
    uint8_t payload[1024];
    ```

    收到：

    ```text
    Length = 60000
    ```

    如果你相信对端：

    ```c
    memcpy(payload, src, length);
    ```

    那就是严重 bug。

2. 防止 Parser 被卡死

    假设，由于噪声：

    ```text
    Length = 65535
    ```

    Parser 可能开始等：

    ```text
    65535 Byte
    ```

    而真正下一帧：

    ```text
    AA 55 ...
    ```

    一直被错误当 Payload。

    于是通信长时间无法恢复。

3. 防止资源攻击

    如果协议跑在 TCP 上，恶意对端不断发：

    ```text
    Length = 超大值
    ```

    你可能：

    - 分配巨大内存
    - 消耗 RAM
    - 卡住任务
    - 导致 DoS

**MIN\_LENGTH** ：

假设 *length* 包含一些固定字段，那么它就必须有长度

### 三、`sizeof(struct)` ≠ Length

假设：

```c
typedef struct
{
    uint16_t magic;
    uint16_t length;
    uint8_t  command;
    uint8_t  payload[256];
    uint16_t crc;
} frame_t;
```

可能想这样：

```c
frame.length = sizeof(frame_t);
```

这是非常危险的思路。

因为：

```text
sizeof(frame_t)
```

描述的是：**C Struct 在当前编译器、当前 ABI、当前内存中的大小。**

而协议 Length 描述的是：**线上 Byte Stream 的协议长度。**

### 四、建议区分三个“长度”

假设 **Length = Command \+ Payload**

```text
+--------+--------+---------------------+-------+
| Magic  | Length |        Body         | CRC16 |
+--------+--------+---------------------+-------+
                  | Command | Payload |
```

区分：

```c
body_len       = Command + Payload
payload_len    = body_len - 1
frame_len      = 固定开销 + body_len
```

这样代码会非常清楚。

---

## 第 5 章 · 第 5 课时

到目前为止，我们的 Frame 还比较“原始”，当前协议：

```text
+--------+--------+---------+---------+-------+
| Magic  | Length | Command | Payload | CRC16 |
+--------+--------+---------+---------+-------+
```

它已经能够解决：

```text
帧从哪里开始？
帧有多长？
执行什么命令？
携带什么数据？
数据有没有损坏？
```

已经可以做很多简单设备通信了。

但是一旦系统稍微复杂一点，就会出现几个新问题。

### 一、Command 只能告诉你“干什么”

例如：

```cpp
CMD = 0x20

// 我们定义：0x20 = SET_SN
```

Host 发给设备：

```text
AA 55 ... 20 ...
```

设备执行以后，要回复什么，怎么让主机知道是成功还是失败

所以我们需要：**Status**

Status 可以理解为：**当前命令的执行结果。**

例如定义：

```text
0x00 = SUCCESS
0x01 = INVALID_COMMAND
0x02 = INVALID_LENGTH
0x03 = INVALID_PARAMETER
0x04 = BUSY
0x05 = CRC_ERROR
0x06 = INTERNAL_ERROR
```

Status 应该放在哪里，没有唯一答案。

一种方式：

```text
Response Body:
    Command
    Status
    Payload

+---------+--------+---------+
| Command | Status | Payload |
+---------+--------+---------+
```

那么响应：

```text
20 00 ...
```

表示：

```text
20 = SET_SN
00 = SUCCESS
```

这是很常见也很好理解的设计。

### 二、接下来第二个问题：同时有多个请求怎么办？

假设 Host 连续发送三个请求：

```text
GET_TEMP
GET_VERSION
GET_STATUS
```

设备陆续回复：

```text
Response A
Response B
Response C
```

实中可能有：

- 异步处理
- 重试
- 超时
- 多线程
- 网络延迟
- 请求乱序
- 重复响应

于是 Host 会问：**这个 Response 到底对应我之前的哪一个 Request？**

这就需要：**Sequence**

*Sequence* 常见名字：

```text
Sequence
Sequence Number
Seq
Transaction ID
Request ID
Message ID
```

核心作用：**给一次请求一个编号，让响应能够和请求对应起来。**

例如：

Host 发：

```text
SEQ = 0x1234
CMD = GET_TEMP
```

设备回复：

```text
SEQ = 0x1234
CMD = GET_TEMP
Status = SUCCESS
Payload = ...
```

Host 一看：SEQ = 0x1234

就知道，这是之前 0x1234 请求的响应。

Sequence 还有一个非常重要的用途：重试去重

假设 Host 发：

```text
SEQ = 100
CMD = ADD_USER
```

设备其实执行成功了，但是响应丢了。

Host 超时后重试：

```text
SEQ = 100
CMD = ADD_USER
```

如果设备完全不看 Sequence，它可能：再执行一次 ADD\_USER

这就可能产生重复操作。

如果设备识别：

```text
这个 SEQ=100 我刚处理过
```

就可以：

```text
不重复执行
直接返回之前结果
```

当然真正做“幂等/去重”还需要更多设计，但 Sequence 是基础。

### 三、协议以后升级怎么办？

假设现在协议 v1：

```text
Magic
Length
Seq
Command
Payload
CRC
```

一年以后你想增加：

```text
Flags
Timestamp
新的 CRC 算法
新的 Command 编码
```

结果旧设备根本不知道怎么解析。

于是需要：**Version**

Version 用来表示：**当前 Frame 使用哪个版本的协议格式。**

例如，设备收到：

```text
Version = 2
```

就可以选择：

```text
按照 v2 解析
```

如果不支持：

```text
UNSUPPORTED_VERSION
```

直接拒绝。

Version 应该放得比较靠前

因为 Parser 越早知道版本越好。

比如：

```text
Magic
Version
Length
...
```

Parser：

```text
找到 Magic
↓
读 Version
↓
判断是否支持
↓
再继续解析
```

如果：不支持，那么可以尽早拒绝。

所以 Version 经常在固定 Header 前部。

### 四、现在我们把协议升级一下

```text
+--------+--------+---------+---------+-------+
| Magic  | Length | Command | Payload | CRC16 |
+--------+--------+---------+---------+-------+
```

现在升级成：

```text
+--------+---------+--------+------+---------+---------+-------+
| Magic  | Version | Length | Seq  | Command | Payload | CRC16 |
+--------+---------+--------+------+---------+---------+-------+
```

- `Length = Seq + Command + Payload`

也就是说：Body = Seq \+ Command \+ Payload

把 Sequence 也放进 Body，只是为了让当前教学协议保持：

```text
Magic
Version
Length
Body
CRC
```

这种整齐结构。

### 五、Request 和 Response 怎么区分？

这是另一个重要设计问题，最常见有三种。

方案 1：通过方向区分

如果链路天然是：

```text
Host → Device = Request
Device → Host = Response
```

那么不一定需要额外字段。

例如：串口主从协议。

Host 收到来自设备的数据，自然认为是 Response。

这是最简单的方法。

方案 2：Command 最高位表示 Request / Response

例如：

```text
Request:
CMD = 0x10

Response：
CMD = 0x90
```

也就是：

```text
Response CMD = Request CMD | 0x80
```

这种做法也很常见。

优点：简单

缺点：Command 编码空间被占掉一部分

方案 3：增加 Type / Flags 字段

例如：

```text
Type:
0x00 = Request
0x01 = Response
0x02 = Event
```

于是 Frame：

```text
Magic
Version
Type
Length
Seq
Command
...
CRC
```

这样更清晰。

尤其当设备还会主动上报：

```text
Event
Notification
Alarm
```

时非常好用。

并不是所有通信都是：

```text
Host 请求
↓
Device 回复
```

设备可能主动发送：

```text
温度过高
防拆报警
按键事件
密钥删除
设备掉电
状态变化
```

这种消息没有对应 Request。

所以它不是：Response

而是：

```text
Event
```

Sequence 对 Event 怎么处理？

有很多方案，例如规定：

```text
Request/Response:
SEQ 正常递增
```

Event：

```text
SEQ = 0
```

或者设备自己的 Event Sequence：

```text
SEQ 独立递增
```

这都可以。

重点仍然是：**协议文档必须规定清楚。**

### 六、Status

1. **Status 和错误信息可以同时存在**

    ```text
    Status = INVALID_PARAMETER
    ```

    Payload 还可以进一步说明：

    ```text
    错误字段编号
    错误原因
    允许范围
    ```

    例如：

    ```text
    CMD     = SET_TIME
    Status  = 0x03 INVALID_PARAMETER
    Payload = 0x02
    ```

    规定：

    ```text
    0x02 = month invalid
    ```

    那么 Host 就知道：**月份字段非法。**

2. Status 不应该设计得太随意

    例如不要：

    ```text
    0 = OK
    1 = FAIL

    # Host 根本不知道失败原因。
    ```

    更好的设计：

    ```text
    0x00 SUCCESS

    0x01 UNKNOWN_COMMAND
    0x02 INVALID_LENGTH
    0x03 INVALID_PARAMETER
    0x04 NOT_SUPPORTED
    0x05 BUSY
    0x06 PERMISSION_DENIED
    0x07 INTERNAL_ERROR
    0x08 TIMEOUT
    ```

    这样上位机、日志、调试都舒服很多。

    但是 Status 也不要无限细分，可以分层：

    ```text
    协议层错误
    应用层错误
    设备内部错误
    ```

    例如：

    ```text
    0x01xx 协议错误
    0x02xx 参数错误
    0x03xx 设备状态错误
    0x04xx 密码运算错误
    ```

    这就是更成熟协议里的错误码体系。

### 七、Command 也应该规划编码空间

建议按功能分类。

例如：

```text
0x01 ~ 0x0F
系统命令

0x10 ~ 0x1F
设备配置

0x20 ~ 0x2F
数据读写

0x30 ~ 0x3F
密码算法

0x40 ~ 0x4F
固件升级
```

这样协议长期维护会清楚很多。

### 八、Version

（1）Version 不等于固件版本

（2）不能每次固件升级都改 Protocol Version

例如：

```text
Firmware 1.0 → Protocol 1
Firmware 1.1 → Protocol 2
Firmware 1.2 → Protocol 3
```

这通常不是好设计。

协议版本应该只在：**线上数据格式或语义发生不兼容变化**** **时升级。

### 九、兼容性通常分两类

1. 向后兼容

    新设备还能理解旧协议。

    ```text
    Device v2
    supports protocol v1 + v2
    ```

2. 向前兼容

    旧设备面对新字段时不会直接崩溃。

    例如通过：

    ```text
    Length
    Version
    TLV
    Reserved
    ```

    等方式实现一定扩展能力。

    这一部分我们后面第 6 章讲序列化和 TLV 时会更深入。

    Reserved 字段是什么，真实协议里还经常看到：

    ```text
    Reserved
    ```

    例如：

    ```text
    Flags     1 Byte
    Reserved  2 Byte
    ```

    意思是：**现在暂时不用，但预留给以后扩展。**

    通常规定：

    ```text
    发送方填 0
    接收方忽略或检查为 0

    # 这样以后可以把 Reserved 某些 bit 拿来做新功能。
    # Reserved 也不是越多越好。否则白白浪费带宽。
    ```

    Flags 一般是一组 bit。

    例如 1 Byte：

    ```text
    bit7 bit6 bit5 bit4 bit3 bit2 bit1 bit0
    ```

    定义：

    ```text
    bit0 = Response
    bit1 = Ack Required
    bit2 = Encrypted
    bit3 = Compressed
    bit4 = Fragmented
    ```

    例如：

    ```text
    Flags = 0x05
    ```

    二进制：

    ```text
    00000101
    ```

    表示：

    ```text
    bit0 = 1
    bit2 = 1
    ```

    也就是：

    ```text
    Response
    +
    Encrypted
    ```

    这是 Bit / Byte 那一章知识真正进入协议设计的地方。

### 十、现在一个“成熟一些”的 Frame 已经出现了

可以设计为：

```c
+--------+---------+-------+--------+------+---------+---------+-------+
| Magic  | Version | Flags | Length | Seq  | Command | Payload | CRC16 |
+--------+---------+-------+--------+------+---------+---------+-------+

/*
例如：
    Magic      2
    Version    1
    Flags      1
    Length     2
    Sequence   2
    Command    1
    Payload    N
    CRC16      2
```

是这里有一个设计哲学，协议不是字段越多越高级。

如果你的设备只是：

```text
MCU A
↕
MCU B
```

支持三个固定命令：

```text
SET_LED
GET_KEY
RESET
```

那么：

```text
Magic
Length
CMD
Payload
CRC
```

已经足够。

**协议字段必须为需求服务**

可以这样判断：

```c
Command
// 几乎一定需要。

Status
// 如果有 Request / Response，通常值得有。

Sequence
// 如果需要请求响应匹配、重试、并发，值得有。

Version
// 如果协议预计长期演进，值得有。

Flags
// 如果存在多个布尔属性，值得有。
```

---

## 第 5 章 · 第 6 课时

> Payload 到底怎么编码：整数、字符串、数组、Struct 在线上应该怎么放？
>

我们已经知道 Frame 可以长这样：

```text
Magic
Version
Length
Sequence
Command
Payload
CRC
```

但这里一直隐藏着一个问题：

> **Payload 里面到底放什么？**
>

例如业务上要发送：

```text
温度 = 25
设备 ID = 0x12345678
名字 = "ABC"
状态 = 1
```

最后在线上必须变成：

```text
XX XX XX XX XX XX ...
```

也就是：**Byte Sequence**

所以真正要解决的是：

> **如何把程序里的数据类型，变成明确、唯一、跨设备一致的 Byte 序列。**
>

这其实已经开始触碰：**序列化 Serialization**

但这一章我们先站在 **Frame Payload** 的角度理解。

### 一、先建立最重要的原则

假设 C 中有：

```c
uint32_t value = 0x1234;
```

变量 `value` 是：程序里的一个整数

而 UART/TCP 真正发送的是：

```text
Byte
Byte
Byte
Byte
```

所以你必须规定：

```text
这个 uint32_t 在线上到底是：

12 34

还是：

34 12
```

这就是：**Wire Format（ ****线上格式。****）**

要把两个世界分开：

```text
程序内部表示    #（如果是小端内存里面就是 34 12，此时没有序列化，数据就是原始电信号）
        ↓
   Serialize   #（读取内存中的数据，调用序列化函数 htons，放入发送缓冲区）
        ↓
协议 Wire Format #（只是一个协议，双方约定好的，不搬运也不修改数据）
        ↓
 UART / TCP     #（比特流搬运，负责把字节变成电平信号或网络包，发出去）
```

接收端：

```text
Wire Format
        ↓
  Deserialize
        ↓
程序内部表示
```

### 二、大小端问题

例如：

```c
uint16_t value = 0x1234;

memcpy(payload, &value, sizeof(value));
```

如果 CPU 是小端，内存中就是：

```text
34 12
```

于是你就发出了：

```text
34 12
```

但协议规定的是大端：

```text
12 34
```

所以：**内存里的 Byte 顺序，不等于协议规定的 Byte 顺序。**

可以将 *Deserialize/serialize* 封装成函数（也就是大小端转换）

```c
static void put_u16_be(uint8_t *buf, uint16_t value)
{
    buf[0] = (uint8_t)(value >> 8);
    buf[1] = (uint8_t)value;
}

static uint16_t get_u16_be(const uint8_t *buf)
{
    return ((uint16_t)buf[0] << 8) |
           ((uint16_t)buf[1]);
}
```

同样可以有：

```c
put_u32_be()
get_u32_be()
put_u64_be()
get_u64_be()
```

以后协议代码会干净很多。

### 三、signed / float 数

例如：

```c
int16_t temperature = -123;
```

协议可以规定：

```c
Temperature:
signed 16-bit integer
two's complement
big-endian
unit = 0.1°C    // 意味着 1 个 LSB（最低有效位）代表 0.1°C。【单位】

// 这句话就已经把线上格式说明得非常清楚了
```

那么：

原码：\-123 \-\-\> 1000 000 0111 1011

反码：1111 1111 1000 0100

补码：1111 1111 1000 0101 \-\-\> **0xFF85**

大端发送：

```text
FF 85
```

接收端按照：*signed int16* 解释

得到：\-123  *\(1000 000 0111 1011\)*

单位是 0\.1°C，即得到

```text
-12.3°C
```

协议里最好尽量避免直接发 float，例如：

```c
float temperature = 25.3f;
```

当然可以直接发送它的 4 Byte 内存表示，但这样会涉及：

- IEEE 754
- 大小端
- 平台实现
- NaN
- Infinity
- 不同语言处理
- 精度问题

很多 MCU 协议更喜欢使用：**定点整数**

例如规定：

```text
Temperature:
int16_t
unit = 0.1°C
```

那么：25\.3°C

编码：

```text
253

即：0x00FD
```

线上：

```text
00 FD
```

简单、明确、稳定。

### 四、字符串

现在假设：

```c
char sn[] = "ABC123";
```

字符串的本质，ASCII：

```text
41 42 43 31 32 33
```

所以 Payload 完全可以是：

```text
41 42 43 31 32 33
```

但字符串有一个问题，接收端怎么知道字符串有多长？

方案 1：固定长度字符串

规定：

```text
SN = 16 Byte
```

那么：**"ABC123"**

可以编码：

```text
41 42 43 31 32 33 00 00 00 00 00 00 00 00 00 00
```

也就是：

```text
有效字符 + Padding
```

优点：

```text
简单
固定位置
```

缺点：浪费空间

方案 2：长度 \+ 字符串

例如：

```text
StringLength = 6
String       = ABC123
```

编码：

```text
06 41 42 43 31 32 33
```

也就是 1Byte 长度，后面跟着 字符串数据

这里的长度**：编码后的 Byte 数量**

方案 3：以 '\\0' 结束

C 字符串习惯：

```text
"ABC123\0"
```

编码：

```text
41 42 43 31 32 33 00
```

然后接收端找到：00

就是结束，但是这种方式一般不如 **Length \+ Data** 稳妥，因为：

- 必须扫描
- 长度上限要额外限制
- 二进制数据本身不能这样处理

### 五、数组

例如：

```c
uint16_t samples[3] =
{
    0x1122,
    0x3344,
    0x5566
};
```

协议规定：

```text
每个 sample = uint16_t big-endian
```

那么 Payload：

```text
11 22
33 44
55 66
```

即：

```text
11 22 33 44 55 66
```

注意：数组不是简单把内存 `memcpy()` 出去。

因为每一个多字节元素都可能需要字节序转换。

**如果数组长度可变怎么办？**

例如：

```text
count = 3
samples = ...
```

可以设计：

```text
+-------+------------------+
| Count | Samples          |
+-------+------------------+
| 1 B   | Count × 2 Byte   |
+-------+------------------+
```

例如：

```text
03 11 22 33 44 55 66
```

这里的 Count 就说明后面有多少 元素

### 六、结构体 Struct

假设我们程序里有：

```c
typedef struct
{
    uint8_t  mode;
    uint32_t device_id;
    uint16_t timeout;
} config_t;
```

很多人第一反应：

```c
config_t cfg;
send(fd, &cfg, sizeof(cfg));
```

在同一个 MCU、同一个编译器里，可能看起来正常工作

但这是我们前面第 3 章一直强调的危险做法。

因为 Struct 是：**内存布局**

Struct 可能有：

```text
Padding
Alignment
CPU Endianness
Compiler ABI
```

所以工业协议应该：**逐字段序列化。**

定义明确 Wire Format，例如：

```text
Config Payload:

Byte 1      Mode
Byte 2~5    Device ID, uint32 big-endian
Byte 6~7    Timeout, uint16 big-endian
```

序列化代码可能这样写

```c
size_t offset = 0;

payload[offset++] = cfg->mode;

put_u32_be(&payload[offset], cfg->device_id);
offset += 4;

put_u16_be(&payload[offset], cfg->timeout);
offset += 2;
```

反序列化也一样

收到：

```text
01 12 34 56 78 00 10
```

解析：

```c
size_t offset = 0;

cfg->mode = payload[offset++];

cfg->device_id = get_u32_be(&payload[offset]);
offset += 4;

cfg->timeout = get_u16_be(&payload[offset]);
offset += 2;
```

所以 Payload 本质上不是“一个 Struct”

更准确地说：

> Payload 是一段由协议定义了字段顺序、字段长度、字节序和编码规则的 Byte Sequence。
>

Payload 可以有自己的“小协议”

外层 Frame：

```text
Magic
Version
Length
Seq
Command
Payload
CRC
```

只负责：Frame 层

而 Payload 里面还可以有结构。

例如：

```text
CMD = SET_CONFIG
```

Payload：

```text
Mode
Timeout
Baudrate
NameLength
Name
```

也就是说：

```text
Frame
└── Payload
    ├── Field 1
    ├── Field 2
    ├── Field 3
    └── ...
```

所以：**Frame 解决消息边界，Payload 解决业务数据表达。**

### 七、Offset 思维

解析 Payload 时，经常维护：

```c
size_t offset = 0;
```

例如：

```text
Payload:
01 00 10 11 22 ...
```

解析：

```c
mode = payload[offset++];
```

然后：

```c
data_len = get_u16_be(&payload[offset]);
offset += 2;
```

再检查：

```c
if (offset + data_len > payload_len)
{
    error();
}
```

然后才访问：

```c
data = &payload[offset];
offset += data_len;
```

这就是一个很常见的：Binary Decoder 基本模式。

可以抽象出 Remaining Length

假设：

```c
payload_len = 20;
offset = 8;
```

剩余：

```text
20 - 8 = 12 Byte
```

所以：

```c
remaining = payload_len - offset;
```

如果下一个字段声明：**len = 16 **直接拒绝。

这种：

```text
offset + len <= payload_len
```

的思想以后会反复用到。

但 `offset + len` 也可能整数溢出

更安全的判断通常是：

```c
if (len > payload_len - offset)
{
    error();
}
```

（ 当然前提先确保：*`offset <= payload_len`** ）*

因为：

```text
offset + len
```

理论上存在溢出风险。

这已经是工业级解析代码的细节了。

### 八、字符串在 Payload 中不要默认是 C 字符串

假设收到：

```text
41 42 43
```

它表示：

```text
"ABC"
```

但它没有：\\0

所以不能直接：*`printf("%s", payload);`*

因为 `%s` 会一直读到：`0x00` 可能越界。

如果协议告诉你：

```text
String Length = 3
```

应该使用长度感知处理，例如：

```c
printf("%.*s", (int)len, (const char *)data);
```

或者自己复制到：

```c
char tmp[len + 1];
```

再补：\\0

嵌入式协议通常优先 Raw Binary，因为：

- 紧凑
- 解析快
- 带宽占用低

Base64 适合：

- 文本协议
- JSON
- XML
- 邮件
- 只能传文本的通道

什么时候 Payload 里适合 Hex String？

主要是 为了人类可读，例如：

- CLI
- 日志
- AT 命令
- 调试接口

### 九、设计一个完整 Command

假设：

```text
CMD = 0x20 = SET_DEVICE_CONFIG
```

业务数据：

```text
mode      uint8_t
device_id uint32_t
timeout   uint16_t
name      UTF-8
```

协议规定：

```text
Payload:
+----------+-----------+---------+----------+--------+
| Mode     | Device ID | Timeout | Name Len | Name   |
+----------+-----------+---------+----------+--------+
| 1 Byte   | 4 Byte BE | 2 B BE  | 1 Byte   | N Byte |
+----------+-----------+---------+----------+--------+
```

具体值：

```text
Mode      = 2
Device ID = 0x12345678
Timeout   = 1000
Name      = "ABC"
```

编码：

```c
Mode： 02

Device ID： 12 34 56 78

Timeout（ 1000 = 0x03E8 ） 即： 03 E8

Name： Length = 3、 ABC = 41 42 43

```

所以 Payload：

```text
02 12 34 56 78 03 E8 03 41 42 43

# Payload Length = 11 Byte
```

如果按照当前教学 Frame

```text
+--------+---------+-------+--------+------+---------+---------+-------+
| Magic  | Version | Flags | Length | Seq  | Command | Payload | CRC16 |
+--------+---------+-------+--------+------+---------+---------+-------+
                                    |         Body             |
```

Body：

```text
Sequence 2
Command  1
Payload 11

# Body Length = 14
```

即：**Length = 0x000E**

假设：

```text
Version = 1
Sequence = 0x002A
CMD = 0x20
```

Frame：

```text
AA 55
01
00 0E
00 2A
20
02 12 34 56 78 03 E8 03 41 42 43
XX XX
```

合起来：

```text
AA 55 01 00 0E 00 2A 20
02 12 34 56 78 03 E8 03 41 42 43
XX XX
```

现在这个 Frame 已经非常接近真实工业通信协议了。

---

## 第 5 章 · 第 7 课时

> Checksum / CRC 到底在检查什么？为什么 CRC 不是加密，也不是身份认证？
>

### 一、终于轮到 Frame 最后那个 CRC

前面我们一直写：

```text
AA 55 ... Payload XX XX
```

最后：**XX XX，**就是 CRC16。

现在的问题是：CRC 到底在干什么？

先看一个最简单的场景。

发送端准备发：

```text
10 11 22 33
```

但是由于：

- 电气干扰
- 线路噪声
- 时序异常
- 存储错误
- DMA / Buffer 异常
- 某个 bit 翻转

接收端实际收到：

```text
10 11 22 32
```

如果没有任何校验：

> 接收端可能完全不知道数据已经变了。
>

于是就需要：**Error Detection 错误检测。**

### 二、****Checksum** 与** CRC

本质上都是：

```text
Data
 ↓
某种算法
 ↓
Check Value

# 然后发送：
Data + Check Value
```

接收端：

```text
收到 Data
 ↓
重新计算
 ↓
得到自己的 Check Value
```

再与接收到的校验值比较。

#### Checksum

把所有 Byte 相加，只保留低 8 bit。

例如：

```text
Data:
10 20 30
```

计算：

```text
0x10 + 0x20 + 0x30
=
0x60
```

所以：**Checksum = 60**

发送：

```text
10 20 30 60
```

接收端重新计算：

```text
10 + 20 + 30 = 60
```

对比接受到的，一致则 OK

显然 **Checksum**  错误检测能力有限。

- **Byte** 顺序交换
- 数据变了但和没变

就无法判断了

#### CRC

```text
Cyclic Redundancy Check

# 循环冗余校验。
```

原理就是，把整段 bit 数据看成一个：

> 二进制多项式
>

然后进行一种特殊的模 2 除法运算。

最后得到一个固定宽度的余数：

```text
CRC8    8 bit
CRC16   16 bit
CRC32   32 bit
```

这个余数就是：**CRC Value**

现在我们只需要理解：

```text
整个 Byte Sequence
       ↓
CRC Algorithm
       ↓
一个固定长度的“特征值”
```

它对：

- bit 的位置
- bit 的排列
- 连续错误
- 多 bit 翻转

这些的检测能力较好

所以 **CRC** 在通信协议里极其常见

CRC16 并不都是一样的，其有很多变体：

```text
CRC-16/IBM
CRC-16/MODBUS
CRC-16/CCITT-FALSE
CRC-16/XMODEM
CRC-16/KERMIT
```

它们虽然都是：

```text
16 bit CRC
```

但结果可能完全不同。

所以只写：**CRC16，**远远不够。

协议文档必须明确：到底是哪一种 CRC16。

### 三、我们当前协议

为了统一，先规定：

```c
Magic      = AA 55
Version    = 1 Byte
Length     = 2 Byte Big Endian
Sequence   = 2 Byte Big Endian
Command    = 1 Byte
Payload    = N Byte
CRC16      = 2 Byte

// CRC Coverage:
// 从 Version 开始，到 Payload 最后一个 Byte
```

即：

```text
Version
Length
Sequence
Command
Payload
```

参与 CRC。

为什么先不把 Magic 算进去，主要为了让逻辑更清晰：

```text
Magic
↓
用来同步

Version ~ Payload
↓
真正的协议内容

CRC
↓
检查协议内容
```

这样容易理解。

### 四、完整流程

假设准备发送：

```text
Version = 01
Length  = 00 05
Seq     = 00 12
CMD     = 10
Payload = 11 22
```

待校验区域：

```text
01 00 05 00 12 10 11 22
```

计算 **CRC16\(\.\.\.\)** ，假设得到：*9A BC*

那么完整 Frame：

```text
AA 55 01 00 05 00 12 10 11 22 9A BC
```

接收端，Parser：

```text
1. 找 Magic
2. 读取 Length
3. 收完整 Frame
4. 提取 received_crc = 0x9ABC
5. 对 Version ~ Payload 重新计算
6. 得到 calculated_crc
7. 比较
```

如果：

```text
calculated_crc == received_crc
```

则：**CRC OK**

### 五、CRC 理解

CRC 本质仍然只是：**错误检测码。**

不同数据理论上可能得到同样的 CRC，这叫：

```text
Collision
```

所以 **CRC OK**  不能代表数据一定没问题，但大多数情况下这就是最高效的方法

CRC 也不能防止别人恶意修改数据

假设攻击者知道你使用：

```text
CRC16
```

原始 Frame：

```text
CMD = SET_LEVEL
Payload = 01
CRC = xxxx
```

攻击者改成：

```text
Payload = FF
```

然后重新计算 CRC，得到：新的合法 CRC

此时接受端也会确认这个 新的合法 CRC

CRC 解决的是“意外错误”，比如：

```text
噪声
bit 翻转
线路干扰
存储损坏
```

而不是：

```text
恶意篡改
伪造
身份冒充
```

#### CRC 不是加密

加密解决的是，**别人能不能看懂数据，**例如：

```text
Plaintext
↓
Encrypt
↓
Ciphertext
```

#### CRC 不是 Hash

Hash*（SHA\-256、SM3）* ，目标是：

```text
固定长度摘要
强抗碰撞
抗原像
抗第二原像
```

二者的功能是一样的，暂时可以理解：

> Hash 是能力更优秀的 CRC
>
> 但是二者的设计初衷不一样
>
> CRC 更高效，应用场景不一样
>

#### CRC 也不是 MAC

MAC：完整性 \+ 身份认证

```text
Message Authentication Code
```

核心是：**使用一个双方共享的秘密密钥，对消息做认证。**

例如：

```text
HMAC-SHA256
CMAC
GMAC
```

或者国密场景中的某些基于密钥的消息认证方案。

MAC 能回答：

> 这条消息是否可能来自持有正确密钥的一方？
>

CRC 完全做不到。

#### 数字签名例如：

```text
SM2 Sign
ECDSA
RSA-PSS
Ed25519
```

大体过程：

```text
Message
+
Private Key
↓
Signature
```

接收方：

```text
Message
+
Signature
+
Public Key
↓
Verify
```

它用于证明：

- 消息完整性
- 签名者身份
- 对方没有对应私钥时无法伪造

这和 CRC 又是完全不同的层次。

### 六、不同的场景适合不同的方式

数据是不是因为线路噪声传坏了？

可以使用：

```text
CRC
```

数据内容有没有被恶意修改？

需要：

```text
MAC

或者

Digital Signature
# 等密码学完整性机制。
```

这条消息到底是不是合法设备发来的？

需要：

```text
MAC

或者

Digital Signature
# 加上密钥管理体系。
```

别人能不能看到 Payload 内容？

需要：

```text
Encryption
```

**所以“加密 \+ CRC”也不是身份认证**

假设：

```text
Payload
↓
SM4 Encrypt
↓
Ciphertext
↓
CRC
```

已经加密了，又有 CRC

但CRC 仍然不是认证，具体的安全性取决于：

- 加密模式
- 是否使用认证加密
- IV/Nonce 处理
- 密钥管理
- 重放保护

如果需要机密性和完整性，现代设计通常更倾向：

```text
AEAD

# Authenticated Encryption with Associated Data
```

暂时了解即可

### 七、CRC 和错误纠正又不是一回事

CRC 通常只能：**发现有错误。**

它一般不能告诉你：

```text
到底哪个 bit 错了
```

更不能自动恢复原值。

错误后一般就是：丢弃 \+ 请求重传

```text
CRC + Timeout + Retry

# 因为 CRC 错误就说明从此传输不可信，等待重传即可
```

经常共同构成基本可靠通信机制。

而能够从冗余信息中修复错误的：

```text
ECC
FEC
Reed-Solomon
Hamming Code
```

属于：**Error Correction**

### 八、Checksum 什么时候还值得用？

对于一些：

- 极简单 MCU
- 极短数据
- 历史协议
- 已经有其他底层校验机制的场景

仍然用 Checksum 更合适

协议设计不是：

> 永远选最复杂的东西。
>

而是：根据错误模型和系统要求选择足够的机制。

### 九、TCP 与 UART

**TCP**

TCP 本身已经有：

> - checksum
>
> - 重传
>
> - 顺序保证
>
> - 丢包恢复
>
> 所以对于很多纯 TCP 应用协议：**完全可以不额外增加 CRC。**
>

但如果业务需要检测：

> - 应用层数据构造错误
>
> - 存储损坏
>
> - 中间软件 bug
>
> - 跨链路转发后的端到端完整性
>
> 也可能额外设计校验。
>

**UART **

UART 硬件一般能够检测一些物理层/字符级问题，例如：

```text
Parity Error
Framing Error
Overrun Error
```

如果开启奇偶校验，还能提供很有限的 bit 错误检测。

但它不知道：

```text
整个 100 Byte Frame 是否完整正确
```

因此应用层 Frame CRC 仍然非常常见。

### 十、CRC 应该在什么时候验证？

正确顺序通常是：

```text
找到 Magic
↓
读取 Length
↓
验证 Length 合法
↓
等待完整 Frame
↓
计算 CRC
↓
CRC OK？
↓
再进入 Command 业务处理
```

这实际上建立了 Parser 的“信任边界”

可以这样理解：

```text
Byte Stream
↓
完全不可信

Magic Found
↓
只是候选 Frame

Length Valid
↓
结构稍微可信

Frame Complete
↓
只是完整候选

CRC Valid
↓
传输完整性基本通过

Command / Payload Validation
↓
协议语义合法

Dispatch
↓
才允许进入业务层
```

这个层次感非常重要。

一个成熟一点的接收流程

以后我们的 Parser 会逐渐变成：

```text
RX Bytes
   ↓
SEARCH_MAGIC
   ↓
READ_FIXED_HEADER
   ↓
CHECK_VERSION
   ↓
PARSE_LENGTH
   ↓
CHECK_LENGTH_RANGE
   ↓
WAIT_FULL_FRAME
   ↓
CHECK_CRC
   ↓
PARSE_BODY
   ↓
CHECK_COMMAND
   ↓
CHECK_PAYLOAD_FORMAT
   ↓
DISPATCH
```

这已经很接近真正的工业级处理管线。

---

## 第 5 章 · 第 8 课时

流式 Frame Parser：接收缓冲区、找帧头、判长度、取完整帧

### 一、为什么必须要有 **Recvive Buffer**

假设一帧长这样：

```text
+--------+--------+--------+--------+---------+--------+
| 0xAA   | 0x55   | Length | CMD    | Payload | CRC    |
+--------+--------+--------+--------+---------+--------+
   1B       1B       1B      1B      N Byte     1B

# 例如：
AA 55 03 01 11 22 33 7A
# Length = Payload_len
```

但 UART 第一次只收到：

```text
AA 55 03
```

第二次收到：

```text
01 11
```

第三次：

```text
22 33 7A
```

所以我们需要一个长期存在的缓存区：

```c
uint8_t rx_buffer[256];
size_t rx_len;
```

底层每收到一些数据先 **append** 进去，Buffer：

```text
AA 55 03 01 11 22 33 7A
```

现在 Parser 才终于拥有一整帧。

这里就会遇到两个问题：

> - 对端发的太快，接收buff 溢出？
>
> - 什么时候将 接收buff 里面的数据交给 Parser 状态机？
>

这里就需要考虑：空闲中断、DMA双缓冲 等等

但是现在我们先不考虑这些，先看了**Parser 状态机**

### 二、Parser 的本质四步

一个最核心的 Frame Parser，可以浓缩为：

```text
① 找 Header
      ↓
② 读取 Length
      ↓
③ 判断数据是否够一整帧
      ↓
④ 校验并取出 Frame
```

然后：

```text
继续解析下一帧
```

这就是流式协议解析器最核心的思想。

#### 第 1 步：找 Header

假设 Buffer 是：

```text
83 19 FF 02 AA 55 03 01 11 22 33 7A
```

前面：

```text
83 19 FF 02
```

可能是：

- 噪声
- 错误数据
- 上一帧损坏后的残留
- 程序中途开始接收造成的残片

Parser 应该寻找：

```text
AA 55
```

前面的：*83 19 FF 02*  全部丢掉

如果只有一个 `AA` 怎么办？

例如 Buffer：

```text
31 22 AA
```

找不到：`AA 55`

前面的 **31 32** 可以丢掉

但是最后的 **AA** 应该保留

#### 第 2 步：判断固定头是否完整

找到：

```text
AA 55
```

之后不可以马上读：**`length `**`= buffer[2];`

例如当前只有：

```text
AA 55
```

这时访问：`buffer[2]`，直接越界

所以必须先判断：

```c
if (rx_len < 3)
{
    return NEED_MORE_DATA;
}
```

因为至少需要：**AA 55 LEN，**才能知道这帧到底多长。

这是一条极其重要的 Parser 编程规则：

> **任何字段在读取之前，都必须先确认 Buffer 中已经存在这些字节。**
>

#### 第 3 步：根据 Length 计算完整 Frame 长度

假设 Buffer：

```text
AA 55 03 01 11
```

现在：`payload_len = buffer[2] = 3;`

那么：`frame_len = 5 + payload_len = 8;`

但是现在：

```text
rx_len = 5
```

所以：**NEED\_MORE\_DATA**

> **NEED\_MORE\_DATA 不是错误。**
>

NEED\_MORE\_DATA 和 ERROR 完全不同

```text
NEED_MORE_DATA
      │
      └── 保留当前数据，等待更多字节


ERROR
      │
      └── 当前候选 Frame 已经不可信，需要恢复同步
```

这是工业级 Parser 最重要的状态之一。

Length 必须做最大值检查

假设收到的 *frame\_len* 是噪声：**FF**

Parser 就可能一直傻等：**260 Byte**

结果后面真正的合法帧：

```text
AA 55 03 ...
```

也全部被它吞进“假 Frame”里面。

于是解析器：**失去同步**

所以工业协议通常都有：

```c
#define MAX_PAYLOAD_SIZE 64

if (payload_len > MAX_PAYLOAD_SIZE)
{
    // 当前 Header 很可能是假 Header
}
```

这不仅仅是“防止数组越界”。

它同时也是：**Frame Synchronization 的重要保护机制。**

#### 第 4 步：检查 CRC

现在假设 Buffer：

```text
AA 55 03 01 11 22 33 7A
```

已经满足：

```text
rx_len >= frame_len
```

说明物理上：**字节数量够了**

下面就需要计算 CRC：

```c
crc_calc = crc8(...);
crc_recv = buffer[frame_len - 1];
```

比较：

```c
if (crc_calc != crc_recv)
{
    return CRC_ERROR;
}
```

只有：

```text
Header 正确
Length 合法
长度完整
CRC 正确
```

才算 **FRAME\_OK**

### 三、整个数据流

假设 UART 分 4 次收到：

```text
第一次：
92 18 AA

第二次：
55 03 01

第三次：
11 22

第四次：
33 7A AA 55 01 02 99 6C
```

第一次

```text
92 18 AA
```

Parser 搜索：虽然没有完整 *AA 55* ，但是最后有 *AA* 有可能是下一帧开头。

因此 Buffer 最终保留：

```text
AA
```

第二次

追加：

```text
55 03 01
```

于是缓冲区：

```text
AA 55 03 01
```

找到 Header，并读取：**LEN = 3**

计算：

```text
frame_len = 8
```

但是当前只有 **4 Byte**

所以：

```text
NEED_MORE_DATA
```

Buffer 不动。

第三次

追加：

```text
11 22
```

得到：

```text
AA 55 03 01 11 22
```

现在：6 \< 8

仍然：**NEED\_MORE\_DATA**

第四次

追加：

```text
33 7A AA 55 01 02 99 6C
```

Buffer：

```text
AA 55 03 01 11 22 33 7A AA 55 01 02 99 6C
```

注意这里有 `Frame 1` \+ `Frame 2`，即：

```text
AA 55 03 01 11 22 33 7A
-----------------------
        Frame 1

AA 55 01 02 99 6C
-----------------
      Frame 2
```

Parser 取走第一帧以后：

```text
AA 55 01 02 99 6C
```

**不能清空 Buffer，**因为这里还有第二帧。

于是 Parser 继续解析，最终 Buffer：*empty*

这就同时解决了：

- 拆包
- 粘包
- 半包
- 多帧
- 噪声
- 帧同步

### 四、这里的设计

Parser 不应该：

```c
parse_one_time();
```

而应该：

```c
while (1)
{
    result = try_parse_frame();

    if (result == FRAME_OK)
    {
        handle_frame();
        continue;
    }

    if (result == NEED_MORE_DATA)
    {
        break;
    }

    if (result == ERROR)
    {
        recover();
        continue;
    }
}
```

请把这一结构记住。

```text
Receive Buffer
                          │
                          ▼
                    try_parse()
                          │
          ┌───────────────┼──────────────┐
          ▼               ▼              ▼
      FRAME_OK       NEED_MORE       ERROR
          │               │              │
      处理一帧           等数据          恢复同步
          │                              │
          └───────────────┐     ┌────────┘
                          ▼     ▼
                        继续解析
```

其中只有：

```text
NEED_MORE_DATA
```

会退出解析循环。

因为：

1. **FRAME\_OK**

    > Buffer 后面可能还有第二帧。
    >
    > 所以继续解析。
    >

2. **ERROR**

    > Buffer 后面可能马上就有一个正确 Frame。
    >
    > 所以恢复同步后继续解析。
    >

3. **NEED\_MORE\_DATA**

    > 是真的没有更多东西可以解析了。
    >
    > 所以退出，等下一批 Byte 到来。
    >

**C 代码骨架**

这一课暂时不写最终完整版，而是先把结构建立起来。

```c
void protocol_feed(const uint8_t *data, size_t len)
{
    /* 1. 新数据追加到接收缓冲区 */
    rx_buffer_append(data, len);

    while (1)
    {
        parser_result_t ret;

        /* 2. 尝试解析一帧 */
        ret = protocol_try_parse();

        switch (ret)
        {
        case PARSER_FRAME_OK:

            /*
             * 已经解析出一帧
             *
             * Buffer 后面可能还有数据，
             * 所以继续 while
             */
            continue;

        case PARSER_NEED_MORE_DATA:

            /*
             * 当前 Buffer 中的数据不足，
             * 等下一次 UART/TCP 数据
             */
            return;

        case PARSER_INVALID_LENGTH:

            /*
             * 当前候选帧不可信
             * 执行同步恢复
             */
            protocol_resync();
            continue;

        case PARSER_CRC_ERROR:

            /*
             * CRC 错误
             * 同样尝试重新同步
             */
            protocol_resync();
            continue;
        }
    }
}
```

现在先不要纠结：

```c
rx_buffer_append()
protocol_try_parse()
protocol_resync()
```

内部到底怎么实现。

这三个函数，正是接下来几课真正要完成的东西。

### 五、特别容易犯的错误

#### 错误 1：每次 UART 收到数据就认为是一帧

```c
HAL_UART_Receive(...);

parse_frame(buf);
```

底层给的是：**Bytes，不是 Frames**

#### 错误 2：Length 读出来后无限相信

```c
frame_len = buf[2] + 5;
```

不检查：**MAX\_PAYLOAD\_SIZE 非常危险**

#### 错误 3：收到完整 Frame 后把 Buffer 清空

例如：

```text
Frame1 | Frame2
```

你处理 Frame1 后：

```c
rx_len = 0;
```

Frame2 就丢了。

#### 错误 4：CRC 错误直接清空整个 Buffer

例如：

```text
坏帧 | 好帧 | 好帧
```

如果看到坏帧之后直接：

```c
rx_len = 0;
```

后面两个好帧也被扔掉。

工业级 Parser 通常应该：**尽可能少地丢数据，并重新寻找 Header。**

---

## 第 5 章 · 第 9 课时

> 工业级 Receive Buffer：`feed()`、`append()`、`consume()` 与滑动缓冲区设计
>

上一课我们已经把 Parser 的主逻辑建立起来了：

```text
Byte Stream
    ↓
Receive Buffer
    ↓
找 Header
    ↓
读 Length
    ↓
判断完整帧
    ↓
CRC
    ↓
取出 Frame
```

这一课专门解决一个非常实际的问题：**这些不断到来的字节，在内存里到底怎么管理？**

也就是：

```text
新数据怎么追加？
解析完一帧后怎么删除？
半帧怎么保留？
噪声怎么丢掉？
Buffer 满了怎么办？
```

### 一、先看最简单的结构

最容易想到的是：

```c
#define RX_BUFFER_SIZE 256

typedef struct
{
    uint8_t data[RX_BUFFER_SIZE];
    size_t  len;
} rx_buffer_t;
```

其中：

```c
data[]    // 存真正收到的字节。

len       // 表示当前有效数据长度。
```

#### append()：新数据追加到尾部

假设当前：

```text
Buffer:

AA 55 03

# len = 3
```

UART 又收到：

```text
01 11 22
```

那么追加后：

```text
AA 55 03 01 11 22
```

对应代码：

```c
memcpy(&buffer->data[buffer->len],
       new_data,
       new_len);

buffer->len += new_len;
```

这就是最基础的：**append\(\)**

#### consume()：删解析完的数据

假设 Buffer：

```text
AA 55 03 01 11 22 33 7A
AA 55 01 02 99 6C
```

Parser 已经成功解析第一帧：

```text
AA 55 03 01 11 22 33 7A
```

剩余：

```text
AA 55 01 02 99 6C
```

最简单的方法就是：

```c
memmove(buffer->data,
        buffer->data + 8,
        buffer->len - 8);
```

然后：

```c
buffer->len -= 8;
```

这就是：

```c
consume(8);
```

这里必须用 `memmove` 而不是 `memcpy`

```c
memcpy(buffer->data,
       buffer->data + count,
       remaining);
```

这里有一个隐患，源区域和目标区域：**发生重叠**

标准 C 中：

```c
memcpy()    // 不能保证重叠区域安全。

memmove()    // 就是专门处理这种情况的。
```

所以这里必须：**memmove\(\)**

#### feed()：底层每来一批字节，就喂给 Parser

例如：

```c
void uart_rx_callback(uint8_t *data, size_t len)
{
    protocol_feed(data, len);
}
```

TCP：

```c
int n = recv(fd, buf, sizeof(buf), 0);

if (n > 0)
{
    protocol_feed(buf, n);
}
```

所以 Parser 根本不关心：

```text
UART
TCP
USB
SPI
DMA
```

它只关心：**给我 bytes**

这其实已经开始体现：**协议层和传输层解耦。**

### 二、一个推荐的接口设计

我们可以先设计：

```c
typedef struct
{
    uint8_t data[RX_BUFFER_SIZE];
    size_t  len;
} rx_buffer_t;
```

然后：

```c
void rx_buffer_init(rx_buffer_t *buffer);

bool rx_buffer_append(rx_buffer_t *buffer,
                      const uint8_t *data,
                      size_t len);

bool rx_buffer_consume(rx_buffer_t *buffer,
                       size_t count);

void rx_buffer_clear(rx_buffer_t *buffer);


// =============== 初始化 ===============
void rx_buffer_init(rx_buffer_t *buffer)
{
    if (buffer == NULL)
    {
        return;
    }

    buffer->len = 0;
}


// =============== 清空 ===============
void rx_buffer_clear(rx_buffer_t *buffer)
{
    if (buffer == NULL)
    {
        return;
    }

    buffer->len = 0;
}
/* 注意一般没有必要：memset(buffer->data, 0, sizeof(buffer->data));
 * 因为有效数据由 len 决定，如果 len=0，那么逻辑上 Buffer 就已经空了。 */
```

### 三、Buffer 满了怎么办？

这是工业代码必须考虑的问题。

```c
#define RX_BUFFER_SIZE 256
```

假设收到的数据 大于 给定的 **RX\_BUFFER\_SIZE**

绝对不能直接：

```c
memcpy(...)
```

否则 数组越界，严重时可能造成：

- 内存破坏
- HardFault
- 栈损坏
- 数据异常
- 安全漏洞

#### 策略 A：整批拒绝

```text
空间不足
→ append 失败
```

优点：

```text
简单
行为确定
```

缺点：

```text
可能损失整批数据
```

#### 策略 B：清空 Buffer，重新同步

例如：

```c
if (overflow)
{
    rx_buffer_clear(buffer);
}
```

然后重新从新数据找 Header。

适用于：

```c
协议允许丢包

/* 例如：
 *    串口控制协议
 *    传感器协议
 *    调试协议
 */
```

#### 策略 C：丢旧数据，保新数据

例如：

```text
旧数据已经长期无法组成合法帧
```

那可以丢掉旧数据。

这类策略比较复杂。

#### 策略 D：使用 Ring Buffer

也就是：

```text
环形缓冲区
```

它可以减少频繁搬移数据。

### 四、不断 `memmove()` 不够完美

按照我们之前的设计：

**consume\(\)：删解析完的数据**

```c
memmove(buffer->data,
        buffer->data + 8,
        buffer->len - 8);

// 然后
buffer->len -= 8;
```

假设我们的 Buffer 每解析一帧：

```c
memmove(...)
```

那么在高吞吐情况下：每秒几千帧

每解析一次：搬一次内存

例如 Buffer 中：

```text
200 Byte
```

取掉前：10 Byte

剩余：

```text
190 Byte
```

就要移动：**190 Byte**

如果频率很高，就浪费 CPU。

所以：`memmove` 方案非常适合简单协议和中低速场景，但并不是最高效的数据结构。

#### 使用：start + len

我们不一定每次都真的移动内存，可以改成：

```c
typedef struct
{
    uint8_t data[RX_BUFFER_SIZE];

    size_t start;
    size_t len;

} rx_buffer_t;
```

现在：

- **start：**表示有效数据从哪里开始。
- **len：**表示有效数据有多少。

这就是“滑动窗口”思想，你可以把 Buffer 想象成：

```text
真实数组：

+--------------------------------------+
| 旧 | 旧 | 旧 | 有 效 数 据 | 空 间      |
+--------------------------------------+
              ↑
            start
```

有效区域：`[start, start + len)`

例如：

```c
start = 20;
len   = 30;
```

那么：`data[20] ~ data[49]`，才是有效数据。

但是这样就会出现一个问题

假设：`RX_BUFFER_SIZE = 100;`

当前：

```c
start = 70;
len   = 20;
```

即有效数据：*70 \~ 89*

数组尾部还剩：*90 \~ 99*

但是数组前面：*0 \~ 69*，已经没用了

这时如果新数据来了 *30 Byte*，逻辑总空间其实完全够

```text
20 + 30 = 50 < 100
↑     ↑
有效  新来 数据
```

但是尾部连续空间只有：10 Byte

这时候有两个选择：

- **Compact**
- **Ring Buffer**

#### Compact：必要时才 memmove

所谓 Compact，就是：**平时不移动，只有尾部空间不够时才整理一次。**

例如：

```text
当前：

[废弃 70B][有效 20B][空 10B]
```

进行 compact 后：

```text
[有效 20B][空 80B]
```

也就是：

```c
memmove(buffer->data,
        buffer->data + buffer->start,
        buffer->len);

buffer->start = 0;
```

这样新来的：*30 Byte 就可以继续追加。*

这比每次 consume 都 memmove 好很多

旧方案：

```text
每消费一次
→ memmove
```

新方案：

```text
consume
→ 只改 start/len

尾部真的没空间
→ 才 memmove 一次
```

性能通常会好很多。

#### 示例代码

可以这样定义：

```c
typedef struct
{
    uint8_t data[RX_BUFFER_SIZE];

    size_t start;
    size_t len;

} rx_buffer_t;
```

**初始化：**

```c
void rx_buffer_init(rx_buffer_t *buffer)
{
    if (buffer == NULL)
    {
        return;
    }

    buffer->start = 0;
    buffer->len   = 0;
}
```

**滑动 Buffer 的 consume：**

```c
bool rx_buffer_consume(rx_buffer_t *buffer,
                       size_t count)
{
    if (buffer == NULL)
    {
        return false;
    }

    if (count > buffer->len)
    {
        return false;
    }

    buffer->start += count;
    buffer->len   -= count;

    if (buffer->len == 0)
    {
        buffer->start = 0;
    }

    return true;
}

/*
为什么空了以后把 start 归零？
假设连续操作后：start = 183;  len   = 0;
虽然逻辑没错，但下一次 append 时比较麻烦。

所以 Buffer 空了之后，即 buffer->len == 0
恢复到标准状态：start = 0;

这样更干净
*/
```

现在就非常简单，完全没有：`memove()`

**append 按照我们之前的逻辑，尾部空间不够时 Compact**

**compact：**

```c
static void rx_buffer_compact(rx_buffer_t *buffer)
{
    if (buffer->start == 0)
    {
        return;
    }

    if (buffer->len > 0)
    {
        memmove(buffer->data,
                buffer->data + buffer->start,
                buffer->len);
    }

    buffer->start = 0;
}
```

**append：**

```c
bool rx_buffer_append(rx_buffer_t *buffer,
                      const uint8_t *data,
                      size_t len)
{
    if (buffer == NULL || data == NULL)
    {
        return false;
    }

    if (len == 0)
    {
        return true;
    }

    /*
     * 总容量首先必须够
     */
    if (buffer->len + len > RX_BUFFER_SIZE)
    {
        return false;
    }

    size_t end = buffer->start + buffer->len;

    size_t tail_space = RX_BUFFER_SIZE - end;

    /*
     * 尾部连续空间不足
     * 但整体空间够
     */
    if (tail_space < len)
    {
        rx_buffer_compact(buffer);    // ⭐⭐⭐

        end = buffer->len;
    }

    memcpy(buffer->data + end,
           data,
           len);

    buffer->len += len;

    return true;
}
```

现在这个设计已经非常实用了。

### 五、封装思想

> 并不是所有的东西都要封装，我们这里的封装是为了理解隔离
>

因为 Parser 最好不要知道底层：

```text
数组怎么移动
start 怎么变化
什么时候 compact
```

Parser 只应该关心：

```text
当前有多少 Byte？
第 i 个 Byte 是什么？
消费前 N Byte
```

那么例如我们读数据，就不应该这样：

```c
buffer->data[buffer->start + i]
```

也就是减少这样的操作：

```c
buffer->start
buffer->len
buffer->data
```

理想的接口是：

```c
size_t rx_buffer_size(const rx_buffer_t *buffer);

uint8_t rx_buffer_at(const rx_buffer_t *buffer,
                     size_t index);

bool rx_buffer_consume(rx_buffer_t *buffer,
                       size_t len);
```

现在把 feed\(\) 补完整

```c
void protocol_feed(const uint8_t *data,
                   size_t len)
{
    if (!rx_buffer_append(&g_rx_buffer, data, len))
    {
        /*
         * Buffer overflow
         *
         * 简化策略：
         * 清空后重新同步
         */
        rx_buffer_clear(&g_rx_buffer);
        return;
    }

    while (1)
    {
        parser_result_t ret;

        ret = protocol_try_parse(&g_rx_buffer);

        if (ret == PARSER_FRAME_OK)
        {
            continue;
        }

        if (ret == PARSER_NEED_MORE_DATA)
        {
            break;
        }

        /*
         * 错误状态
         * Parser 内部或这里做同步恢复
         */
        protocol_resync(&g_rx_buffer);
    }
}
```

### 六、Buffer 和 Parser 的职责一定要分开

这是架构上非常重要的一点。

Receive Buffer：

```text
负责存字节
负责追加
负责消费
负责空间管理
```

Parser：

```text
负责理解协议
负责找 Header
负责 Length
负责 CRC
负责 Frame
```

也就是：

```text
Receive Buffer
    │
    │ 提供 bytes
    ▼
Frame Parser
    │
    │ 提供完整 Frame
    ▼
Command Handler
```

三层不要混成一团。

这对嵌入式尤其重要

因为以后你可能换传输方式：

```text
UART
↓
TCP
```

甚至：

```text
USB CDC
BLE
SPI
RS485
```

Parser 不应该重写。

底层只需要继续：

```c
protocol_feed(data, len);
```

所以好的协议层设计通常是：

```text
Transport Layer
      ↓ bytes
Protocol Buffer
      ↓
Frame Parser
      ↓ frame
Command Dispatcher
```

这就是非常典型的分层设计。

### 六、Ring Buffer

> 既然滑动 Buffer 这么好，为什么还有 Ring Buffer？
>

Ring Buffer：

```text
+----------------------+
|      环形数组         |
+----------------------+
     ↑            ↑
    read         write
```

写到尾部后：重新回到数组开头

所以完全不需要：

```c
memmove()
```

这对：

```text
UART DMA
高速串口
连续采样
高吞吐 TCP
```

尤其有优势。

但是 Ring Buffer 的缺点是：

- Parser 可能遇到：**Frame 被数组尾部截断**

例如：

```text
数组末尾：
AA 55 03 01

数组开头：
11 22 33 7A

但这逻辑上是一帧：
AA 55 03 01 11 22 33 7A
```

但物理内存不是连续的，Parser 写起来更复杂。

当前为什么不直接上 Ring Buffer？

因为这一章重点是：

```text
Frame Protocol Parsing
```

而不是：**高性能队列数据结构**

现在使用：

```text
滑动线性 Buffer
+
必要时 compact
```

已经非常适合学习和绝大多数普通 MCU 协议。

它有一个巨大的优点：

> **有效 Frame 在内存中始终连续。**
>

后面做：

```c
crc(buf, frame_len)
memcpy(frame)
parse fields
```

都很方便。

什么时候才值得用 Ring Buffer？

大概可以这样理解：

**普通 UART 控制协议，例如：**

```text
115200
几十字节一帧
命令交互
```

滑动 Buffer：完全够用

**RS485 / UART 中高速连续流，例如：**

```text
1 Mbps
持续收包
```

可以考虑：Ring Buffer

对于 MCU 场景下的高吞吐串口通信

UART \+ DMA ：

```c
DMA Circular Buffer
+
Software Ring Buffer
```

只有网络高吞吐，可能进一步考虑：

```text
zero-copy            # 避免内存拷贝，直接在内核缓冲区和用户态共享数据。
scatter/gather       # 一次性处理分散的内存块，减少系统调用开销。
ring queue           # 更贴近网卡/驱动的队列机制，适合极高吞吐场景。
```

但是那已经不是我们当前这个课程阶段的核心了。

### 七、一个非常重要的容量关系

假设协议规定：

```c
#define MAX_PAYLOAD_SIZE 128
```

Frame：

```text
Header     2
Length     2
CMD        1
Payload  128
CRC        2
```

那么最大 Frame = **135 Byte**

这时我们通常定义 接收Buffer 大小：

```c
#define RX_BUFFER_SIZE (MAX_FRAME_SIZE * 2)
```

为什么两倍？因为某次接收中可能是：

```text
完整 Frame1
+
完整 Frame2
```

所以 Buffer 容量设计要考虑：

```text
最大帧
+
粘包
+
处理延迟
```

写嵌入式代码时，“不变量”非常重要。

我们的结构必须始终满足：

```c
buffer->start <= RX_BUFFER_SIZE

buffer->len <= RX_BUFFER_SIZE

buffer->start + buffer->len <= RX_BUFFER_SIZE
```

也就是：**有效数据绝不能跑出数组**

如果所有函数都维护这个条件，那么 Parser 就会稳定很多。

### 八、最终推荐结构

现在可以得到：

```c
typedef struct
{
    uint8_t data[RX_BUFFER_SIZE];

    size_t start;
    size_t len;

} rx_buffer_t;
```

接口：

```c
void rx_buffer_init(rx_buffer_t *buffer);

void rx_buffer_clear(rx_buffer_t *buffer);

bool rx_buffer_append(rx_buffer_t *buffer,
                      const uint8_t *data,
                      size_t len);

bool rx_buffer_consume(rx_buffer_t *buffer,
                       size_t len);

size_t rx_buffer_size(const rx_buffer_t *buffer);

uint8_t rx_buffer_at(const rx_buffer_t *buffer,
                     size_t index);
```

内部还有：

```c
static void rx_buffer_compact(rx_buffer_t *buffer);
```

这已经是一个非常像样的接收缓存模块了。

1\. `feed` 底层把新收到的数据交给协议层：

```c
protocol_feed(data, len);
```

2\. `append` 不是覆盖：

```text
旧数据 + 新数据
```

3\. `consume` 已经处理的数据必须从逻辑 Buffer 中删除。

4\. 不要每次 consume 都 `memmove`

可以通过：

```text
start + len
```

实现滑动窗口。

5\. 必要时再 `compact` 只有尾部连续空间不够：

```text
才 memmove
```

6\. Buffer 与 Parser 分层

```text
Buffer
负责 bytes

Parser
负责 protocol
```

一定不要混淆。

### 九、从“会收数据”进入“会管理数据”

很多嵌入式代码只做到：

```c
uart_receive(buf);
```

但真正可靠的通信系统需要再往前一步：

```text
UART
 ↓
Byte Stream
 ↓
Receive Buffer
 ↓
Frame Parser
 ↓
Command
```

到这里，你已经不应该再把：

```c
HAL_UART_Receive()
recv()
```

理解成：**收到一帧**

而是：**收到了一批字节，我把它们加入协议解析器的输入流。**

---

## 第 5 章 · 第 10 课时

Resync：帧同步恢复，错误后到底该丢多少数据？

这一课非常关键，因为一个协议解析器“能不能长期稳定跑”，很大程度上就取决于：

> **出错以后，能不能尽快重新找到下一帧。**
>

前面我们已经有了：

```text
Byte Stream
   ↓
Receive Buffer
   ↓
Parser
```

现在要补上最后一个关键能力：

```text
出错
  ↓
Resync
  ↓
重新找到合法 Header
```

### 一、重新同步

假设协议格式还是：

```text
AA 55 LEN CMD PAYLOAD CRC
```

数据流变成：

```text
92 17 FF AA 55 03 01 11 22 33 7A
```

前面的：

```text
92 17 FF
# 就是垃圾数据。
```

Parser 必须跳过它们，重新找到：**AA 55**

这个过程就是：

> **重新同步，Resynchronization，简称 Resync。**
>

最简单的 Resync：一直找 Header

```text
Buffer
  ↓
从前往后扫描
  ↓
找到 AA 55
  ↓
把 AA 55 前面的全部丢掉
```

这就是最基本的：`sync_to_header()`

### 二、错误以后到底丢多少

CRC 错误后，直接全丢：

```c
if (crc_error)
{
    rx_buffer_clear(buffer);
}
```

看起来很干净，但问题很大。

假设 Buffer：

```text
坏帧 | 好帧

例如：
AA 55 03 01 11 22 33 FF
AA 55 01 02 99 6C
```

第一帧 CRC 错误，如果你直接：

```c
clear();
```

那么第二帧，也一起丢了。

工业 Parser 更合理的原则是：

> **出错时尽量少丢数据。**
>

这里有一个非常重要的经验：

> **通常不要直接丢掉整帧，而是先丢掉当前候选 Header 的第一个字节，再重新扫描 Header。**
>

```text
当前 Header 不可信
    ↓
只丢 1 Byte
    ↓
重新扫描 AA 55

# 注意：
# 为什么只丢 1 Byte，而不是 2 Byte
# 因为有可能 收到的是：AA AA ...
# 第二个 AA 也许是 Header，为了保险只丢 1 Byte
```

没有找到 Header 时，也不能全部清空

例如 Buffer：

```text
31 22 AA
```

没有完整的：AA 55

但最后一个  AA ，可能是下一帧 Header 的第一半。

因此没有找到完整 Header 时，应该：

```text
如果最后一个字节 == 0xAA
    保留它
否则
    全部丢掉
```

### 三、更完整的同步函数

```c
#define HEADER0 0xAA
#define HEADER1 0x55

static bool protocol_sync_to_header(rx_buffer_t *buffer)
{
    size_t len = rx_buffer_size(buffer);

    if (len == 0)
    {
        return false;
    }

    for (size_t i = 0; i + 1 < len; i++)
    {
        if (rx_buffer_at(buffer, i)     == HEADER0 &&
            rx_buffer_at(buffer, i + 1) == HEADER1)
        {
            if (i > 0)
            {
                rx_buffer_consume(buffer, i);
            }

            return true;
        }
    }

    /*
     * 没找到完整 Header。
     *
     * 如果最后一个字节可能是 Header 第一个字节，
     * 就保留它。
     */
    if (rx_buffer_at(buffer, len - 1) == HEADER0)
    {
        rx_buffer_consume(buffer, len - 1);
    }
    else
    {
        rx_buffer_consume(buffer, len);
    }

    return false;
}
```

### 四、Parser 的完整顺序

现在 Parser 应该这样思考。

**第一步：**

先保证 Buffer 开头就是 Header，也就是：

```c
if (!protocol_sync_to_header(buffer))
{
    return PARSER_NEED_MORE_DATA;
}
```

同步以后，Buffer 开头一定是：

```text
AA 55 ...
```

**第二步：固定字段够不够**

我们的协议：

```text
AA 55 LEN CMD ...
```

至少要有：4 Byte

所以：

```c
if (rx_buffer_size(buffer) < 4)
{
    return PARSER_NEED_MORE_DATA;
}
```

**第三步：检查 Length**

```c
uint8_t payload_len = rx_buffer_at(buffer, 2);
```

假设最大：

```c
#define MAX_PAYLOAD_SIZE 64
```

检查：

```c
if (payload_len > MAX_PAYLOAD_SIZE)
{
    return PARSER_INVALID_LENGTH;
}
```

这里一定不要写成：继续等更多数据，因为 Length 已经确定不合法。

Invalid Length 怎么做

最稳妥策略：

```c
rx_buffer_consume(buffer, 1);
```

然后重新执行：

```text
sync_to_header()
```

原则还是：**最小丢弃。**

**第四步：完整帧够不够**

计算：

```c
size_t frame_len = 5 + payload_len;
```

然后：

```c
if (rx_buffer_size(buffer) < frame_len)
{
    return PARSER_NEED_MORE_DATA;
}
```

说明当前数据：

```text
Header 合法
Length 合法
只是还没到齐
```

所以正确动作是：

```text
保留所有数据
等待下一批 bytes
```

**第五步：CRC**

数据够了：

```text
AA 55 LEN CMD PAYLOAD CRC
```

现在检查：

```c
uint8_t recv_crc =
    rx_buffer_at(buffer, frame_len - 1);

uint8_t calc_crc =
    protocol_crc(...);
```

如果：

```c
calc_crc != recv_crc
```

说明当前候选 Frame 不可信，这时：

```c
return PARSER_CRC_ERROR;
```

上层执行：

```c
consume(1);
```

然后重新同步。

CRC 错可能来自：

```text
1. 传输过程中 Bit 错
2. Payload 损坏
3. Header 是假的
4. Length 是误解析的
5. 丢字节
6. 插入字节
```

Parser 并不知道是哪一种。

所以最稳妥的是：不对错误原因做过度假设，只把当前最前面的候选起点判定为“不可信”。

于是：

```text
consume(1)
↓
重新扫描
```

### 五、完整错误恢复流程

```c
while (1)
{
    parser_result_t ret =
        protocol_try_parse(&buffer);

    switch (ret)
    {
    case PARSER_FRAME_OK:
        continue;

    case PARSER_NEED_MORE_DATA:
        return;

    case PARSER_INVALID_LENGTH:
    case PARSER_CRC_ERROR:

        /*
         * 当前候选 Header 不可信。
         *
         * 最小丢弃一个 Byte，
         * 然后重新寻找 Header。
         */
        rx_buffer_consume(&buffer, 1);

        continue;
    }
}
```

注意这里的：

```c
continue;
```

不是出错就退出，而是：

```text
出错
↓
恢复
↓
继续找下一帧
```

#### 成功 Frame 又怎么处理？

如果：

```text
Frame 是经过 CRC 验证的合法 Frame
```

那我们可以非常有信心地：

```c
consume(frame_len);
```

一次把整帧消费掉。

因为它已经被确认合法，所以：

```text
错误 Frame
→ consume(1)

合法 Frame
→ consume(frame_len)
```

这是一个非常重要的对比。

### 六、Parser 核心代码已经可以写出来了

```c
#define HEADER0          0xAA
#define HEADER1          0x55
#define MAX_PAYLOAD_SIZE 64

typedef enum
{
    PARSER_NEED_MORE_DATA,
    PARSER_FRAME_OK,
    PARSER_INVALID_LENGTH,
    PARSER_CRC_ERROR

} parser_result_t;

parser_result_t protocol_try_parse(rx_buffer_t *buffer)
{
    /*
     * **1. 找 Header**
     */
    if (!protocol_sync_to_header(buffer))
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * **2. 至少需要 Header + LEN + CMD**
     */
    if (rx_buffer_size(buffer) < 4)
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * **3. Length**
     */
    uint8_t payload_len =
        rx_buffer_at(buffer, 2);

    if (payload_len > MAX_PAYLOAD_SIZE)
    {
        return PARSER_INVALID_LENGTH;
    }

    /*
     * Header(2)
     * Length(1)
     * CMD(1)
     * Payload(N)
     * CRC(1)
     */
    size_t frame_len =
        5 + payload_len;

    /*
     * **4. 判断完整 Frame 是否到齐**
     */
    if (rx_buffer_size(buffer) < frame_len)
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * **5. CRC**
     */
    uint8_t recv_crc =
        rx_buffer_at(buffer, frame_len - 1);

    uint8_t calc_crc =
        protocol_calc_crc(buffer, frame_len - 1);

    if (recv_crc != calc_crc)
    {
        return PARSER_CRC_ERROR;
    }

    /*
     * **6. Frame 合法**
     */
    protocol_handle_frame(buffer, frame_len);

    /*
     * **7. 消费整个合法 Frame**
     */
    rx_buffer_consume(buffer, frame_len);

    return PARSER_FRAME_OK;
}
```

这已经是整个第 5 章最核心的一段代码之一。

### 七、现在 feed() 也清晰了

```c
void protocol_feed(const uint8_t *data,
                   size_t len)
{
    if (!rx_buffer_append(&g_rx_buffer,
                          data,
                          len))
    {
        /*
         * Buffer overflow
         */
        rx_buffer_clear(&g_rx_buffer);
        return;
    }

    while (1)
    {
        parser_result_t ret =
            protocol_try_parse(&g_rx_buffer);

        switch (ret)
        {
        case PARSER_FRAME_OK:
            /*
             * Buffer 后面可能还有 Frame
             */
            continue;

        case PARSER_NEED_MORE_DATA:
            /*
             * 等下一批 bytes
             */
            return;

        case PARSER_INVALID_LENGTH:
        case PARSER_CRC_ERROR:

            /*
             * 当前候选起点失效
             */
            if (rx_buffer_size(&g_rx_buffer) > 0)
            {
                rx_buffer_consume(&g_rx_buffer, 1);
            }

            continue;
        }
    }
}
```

到这里，Parser 的大框架基本完整了。

### 八、需要理解的点

#### Resync 本质是“容错状态机”

可以把 Parser 看成：

```text
SEARCH_HEADER
     ↓
READ_FIXED_FIELDS
     ↓
CHECK_LENGTH
     ↓
WAIT_FULL_FRAME
     ↓
CHECK_CRC
     ↓
FRAME_OK
```

任何错误：

```text
INVALID_LENGTH
CRC_ERROR
```

都会：

```text
退回 SEARCH_HEADER
```

但不是：全部清空

而是：最小前进

所以本质上它就是一个：**容错的有限状态机。**

#### 一个很重要的“前进性”原则

Parser 必须保证：

> **出错时 Buffer 一定发生变化。**
>

否则容易出现死循环。

例如：

```c
while (1)
{
    ret = parse();

    if (ret == CRC_ERROR)
    {
        continue;
    }
}
```

如果没有：

```c
consume(...)
```

下一轮还是同一批数据：CPU 卡死。

所以一定要牢记：

```text
FRAME_OK
→ consume(frame_len)

ERROR
→ consume(至少 1 Byte)

NEED_MORE
→ return
```

这三类状态各自必须有明确动作。

这其实是 Parser 最重要的循环不变量

每次 while 循环必须满足三者之一：

```text
① 成功消费一整帧

② 出错至少消费一个 Byte

③ 数据不足，退出等待
```

绝不能出现：

```text
既不消费数据
又不退出
```

否则就是死循环，这条规则特别值得记。

#### Buffer Overflow 其实也是一种失去同步

假设：

```text
Buffer 已经满了
```

说明可能存在：

```text
1. 长时间没找到合法 Header
2. Length 异常
3. 上层处理不及时
4. 数据速率过高
5. Buffer 太小
6. 对端持续发送垃圾
```

此时：当前缓存可信度已经很低

所以很多控制类协议会选择：

```c
rx_buffer_clear();
```

然后从新数据重新同步。

这和 CRC\_ERROR 不同：

```text
CRC_ERROR
→ 尽量少丢

BUFFER_OVERFLOW
→ 可以更激进
```

因为 Overflow 已经意味着系统进入异常状态。

### 九、工业实现还会加错误计数器

例如：

```c
typedef struct
{
    uint32_t frames_ok;
    uint32_t crc_errors;
    uint32_t length_errors;
    uint32_t resync_count;
    uint32_t overflow_count;

} protocol_stats_t;
```

每次：

```c
stats.frames_ok++;

或者

stats.crc_errors++;
```

这样以后现场调试特别有价值。

例如设备通信偶发异常，你看到：

```text
frames_ok     = 1234567
crc_errors    = 2
length_errors = 0
overflow      = 0
```

和：

```text
crc_errors    = 50000
overflow      = 2000
```

代表的问题完全不同。

计数对嵌入式排查很重要

因为现场设备往往不能一直挂着逻辑分析仪。

你只能通过：

```text
日志
状态查询命令
诊断接口
```

查看系统发生了什么。

所以一个成熟协议栈经常会提供：

```text
RX frames
TX frames
CRC errors
Length errors
Dropped bytes
Buffer overflows
Resync count
```

这属于：**可观测性。**

也是“工业代码”和“Demo 代码”的重要区别。

### 十、已形成的完整数据链

```text
UART / TCP / USB
        ↓
      feed()
        ↓
     append()
        ↓
Receive Buffer
        ↓
 sync_to_header()
        ↓
    Length
        ↓
   Full Frame?
        ↓
      CRC
        ↓
  ┌─────┴─────┐
  │           │
 OK         ERROR
  │           │
consume      consume(1)
frame_len      │
  │           │
  └─────┬─────┘
        ↓
     continue
```

只有：

```text
NEED_MORE_DATA
```

才：

```text
退出
等待新 bytes
```

这就是一个完整的流式 Frame Parser。

---

## 第 5 章 · 第 11 课时

从 Frame 到 Command：字段提取、Payload 交付与协议层/业务层解耦

前面几课我们已经把最难的“字节流解析”完成了。

现在整个接收链路已经是：

```text
UART / TCP / USB
        ↓
      feed()
        ↓
 Receive Buffer
        ↓
   Frame Parser
        ↓
 找 Header / Length / CRC
        ↓
    合法 Frame
```

但到这里还没结束，因为真正的应用程序并不关心：

```text
AA 55 03 01 11 22 33 7A
```

它真正关心的是：

```text
CMD = 0x01

Payload =
11 22 33
```

所以下面要解决：

> **Parser 得到一个合法 Frame 以后，应该怎样把它变成“命令”，再交给业务层处理？**
>

这一步实际上就是：

```text
Byte Stream
    ↓
Frame
    ↓
Command
    ↓
Application
```

这会把：

```text
Byte Stream
→ Frame
```

继续推进到：

```text
Frame
→ Command
→ Application
```

也就是协议栈从“会解析”走向“真正能用”。

先回顾我们的协议

```text
+--------+--------+--------+--------+---------+--------+
| 0xAA   | 0x55   | Length | CMD    | Payload | CRC    |
+--------+--------+--------+--------+---------+--------+
   1B       1B       1B      1B      N Byte     1B
```

例如：

```text
AA 55 03 01 11 22 33 7A
```

现在 Parser 已经完成：

```text
Header 正确
Length 正确
Frame 完整
CRC 正确
```

那么从业务角度看，真正有意义的是：

```text
CMD     = 01
Payload = 11 22 33
```

Header、Length、CRC 都是协议自身为了传输而存在的。

### 一、Parser 和 业务逻辑 一定不要混在一起

初学时很容易写成这样：

```c
if (crc_ok)
{
    uint8_t cmd = buf[3];

    if (cmd == 0x01)
    {
        gpio_set(...);
    }
    else if (cmd == 0x02)
    {
        flash_write(...);
    }
    else if (cmd == 0x03)
    {
        reboot();
    }
}
```

这段代码能跑，但设计很差，因为：

```text
Frame Parser
```

本来只负责：

```text
找到 Frame
验证 Frame
提取字段
```

现在却又负责：

```text
GPIO
Flash
重启
业务状态
```

协议层和业务层彻底耦合了

以后想：`UART → TCP`或者协议稍微修改，就会牵一大片代码。

正确的分层，推荐形成下面这个结构：

```text
┌───────────────────────────┐
│       Transport Layer     │
│ UART / TCP / USB / RS485  │
└─────────────┬─────────────┘
              │ bytes
              ▼
┌───────────────────────────┐
│      Receive Buffer       │
└─────────────┬─────────────┘
              │ bytes
              ▼
┌───────────────────────────┐
│       Frame Parser        │
│ Header / Length / CRC     │
└─────────────┬─────────────┘
              │ command
              ▼
┌───────────────────────────┐
│    Command Dispatcher     │
│       CMD 分发            │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│     Application Layer     │
│ GPIO / Flash / Crypto ... │
└───────────────────────────┘
```

这是这一课最重要的架构。

### 二、Parser 最终输出

我们可以定义一个：

```c
typedef struct
{
    uint8_t cmd;

    const uint8_t *payload;
    size_t payload_len;

} protocol_message_t;
```

这样一个合法 Frame：

```text
AA 55 03 01 11 22 33 CRC
```

经过 Parser 后：

```c
msg.cmd = 0x01;

msg.payload =
    11 22 33

msg.payload_len = 3;
```

业务层就完全不需要关心：

```text
AA 55
Length
CRC
```

它只看到：“收到命令 `0x01`，附带 3 Byte 参数。”

这才是好的抽象。

字段位置计算，我们的 Frame：

```text
index:

0    1    2    3    4 ......
AA   55   LEN  CMD  PAYLOAD
```

所以：

```c
#define FRAME_HEADER_SIZE  2
#define FRAME_LEN_OFFSET   2
#define FRAME_CMD_OFFSET   3
#define FRAME_PAYLOAD_OFFSET 4
```

那么：

```c
uint8_t payload_len =
    rx_buffer_at(buffer, FRAME_LEN_OFFSET);

uint8_t cmd =
    rx_buffer_at(buffer, FRAME_CMD_OFFSET);
```

Payload 从：**offset = 4** 开始。

### 三、合法 Frame 到底要不要复制

#### 不复制：Zero-copy

假设 Buffer 中现在有：

```text
AA 55 03 01 11 22 33 CRC
```

我们可以直接让：`msg.payload` 指向 **Receive Buffer**：

```c
msg.payload =
    &buffer->data[buffer->start + 4];
```

这就叫：**Zero\-copy，零拷贝。**

即没有：

```c
memcpy()
```

效率很好。

Zero\-copy 存在的问题

假设：

```c
msg.payload
```

直接指向：**Receive Buffer**

然后业务层保存了这个指针：

```c
g_saved_payload = msg.payload;
```

接着 Parser：

```c
rx_buffer_consume(...)

// 或者
compact()

// 又或者新数据
append()
```

**Buffer** 内容可能发生变化，于是：

```c
g_saved_payload
```

指向的内容就不再可靠。

所以 zero\-copy 有一个非常重要的生命周期规则：

> **Payload 指针只在当前 Frame 被消费之前有效。**
>

所以在使用时要：同步处理 \+ Zero\-copy

流程：

```text
Parser
  ↓
构造 message
  ↓
立即调用 handler
  ↓
handler 返回
  ↓
consume(frame)
```

代码：

```c
protocol_handle_message(&msg);

rx_buffer_consume(buffer, frame_len);
```

这样：`msg.payload` 在 `protocol_handle_message()`

执行期间始终有效，这是非常高效的方案。

#### 复制 Payload

如果业务层需要：

```text
异步处理
放入 Queue
跨任务处理
稍后使用
```

那就不能长期保存 **Receive Buffer** 里的指针，就需要复制，例如：

```c
typedef struct
{
    uint8_t cmd;
    uint8_t payload[MAX_PAYLOAD_SIZE];
    size_t payload_len;

} protocol_message_t;
```

然后：

```c
memcpy(msg.payload,
       payload_ptr,
       payload_len);
```

这样消息拥有自己的数据。后面 Parser：

```text
consume
compact
append
```

都不会影响它。

#### 什么时候选 Zero-copy？

如果你的系统是：

```text
收到命令
↓
立即处理
↓
立即回复
```

比如：

```text
读版本
设置 GPIO
获取状态
读寄存器
```

Zero\-copy 很合适。因为：

```text
简单
快
省 RAM
```

#### 什么时候必须 Copy？

例如 FreeRTOS：

```text
UART Task
    ↓
Parser
    ↓
xQueueSend()
    ↓
Worker Task
```

这时候：

```text
Worker Task
```

可能几毫秒后才处理，Receive Buffer 早就已经变了。

所以必须：

```text
Copy
```

或者使用其他明确的内存所有权机制。

因此：**是否复制，本质上取决于数据生命周期和所有权。**

*这是非常重要的软件设计概念。*

为了把核心思想讲清楚，我们现在先采用：

```text
同步处理 + Zero-copy
```

即：

```text
找到合法 Frame
↓
构造 message
↓
立即 dispatch
↓
handler 返回
↓
consume Frame
```

注意顺序：

```text
dispatch

# 一定在：
consume    # 之前
```

### 四、构造 Message

假设有效 Frame 起点是：

```c
uint8_t *frame;
```

那么：

```c
protocol_message_t msg;

msg.cmd = frame[FRAME_CMD_OFFSET];

msg.payload =
    &frame[FRAME_PAYLOAD_OFFSET];

msg.payload_len =
    frame[FRAME_LEN_OFFSET];
```

然后：

```c
protocol_dispatch(&msg);
```

**Command Dispatcher 作用**

> 根据 CMD，把消息送到不同处理函数。
>

例如定义：

```c
#define CMD_GET_VERSION   0x01
#define CMD_SET_LED       0x02
#define CMD_GET_STATUS    0x03
```

然后：

```c
static void protocol_dispatch(
    const protocol_message_t *msg)
{
    switch (msg->cmd)
    {
    case CMD_GET_VERSION:
        handle_get_version(msg);
        break;

    case CMD_SET_LED:
        handle_set_led(msg);
        break;

    case CMD_GET_STATUS:
        handle_get_status(msg);
        break;

    default:
        handle_unknown_command(msg);
        break;
    }
}
```

这就是最基础的：`Command Dispatcher`

Dispatcher 单独一层

因为：

```text
Parser
```

只应该回答：**这是不是一个合法 Frame？**

```text
Dispatcher
```

回答：**这是哪个 Command？**

```text
Handler
```

回答：**这个 Command 到底要干什么？**

三个职责完全不同。

### 五、Handler 应该再次检查 Payload Length

虽然 Frame Parser 已经检查：

```text
Length <= MAX_PAYLOAD
```

但它并不知道：

```text
CMD 0x02
```

到底需要多少参数。

所以：

```c
static void handle_set_led(
    const protocol_message_t *msg)
{
    if (msg->payload_len != 2)
    {
        protocol_send_error(...);
        return;
    }

    uint8_t led_id =
        msg->payload[0];

    uint8_t state =
        msg->payload[1];

    ...
}
```

这里的长度检查属于：**Command 语义校验。**

Parser 校验和 Command 校验不是一回事

**Parser 检查：**Frame Validation

```text
Header 是否正确
Length 是否超过协议最大值
Frame 是否完整
CRC 是否正确
```

**Handler 检查：**Command Validation

```text
这个 CMD 是否允许这个 Length？
参数是否合法？
数值范围是否正确？
当前状态是否允许执行？
```

### 六、一些注意的点

#### Unknown Command 的处理

假设：

```text
CMD = 0x99
```

而 Frame：

```text
Header 正常
Length 正常
CRC 正常
```

那么 Parser 不应该说：

```text
Frame Error
```

它应该正常把 Frame 交给 Dispatcher。

Dispatcher 再判断：

```c
default:
    handle_unknown_command(msg);
```

然后可以回复：

```text
UNSUPPORTED_COMMAND
```

这说明：**未知 Command 不等于非法 Frame，**这一点非常值得记。

#### 返回状态也需要规范化

定义：

```c
typedef enum
{
    PROTOCOL_STATUS_OK            = 0x00,
    PROTOCOL_STATUS_INVALID_CMD   = 0x01,
    PROTOCOL_STATUS_INVALID_LEN   = 0x02,
    PROTOCOL_STATUS_INVALID_PARAM = 0x03,
    PROTOCOL_STATUS_BUSY          = 0x04,
    PROTOCOL_STATUS_INTERNAL_ERR  = 0x05

} protocol_status_t;
```

于是 Handler 返回：

```c
protocol_status_t
```

例如：

```c
static protocol_status_t handle_set_led(
    const protocol_message_t *msg)
{
    if (msg->payload_len != 2)
    {
        return PROTOCOL_STATUS_INVALID_LEN;
    }

    if (msg->payload[0] >= LED_COUNT)
    {
        return PROTOCOL_STATUS_INVALID_PARAM;
    }

    led_set(msg->payload[0],
            msg->payload[1]);

    return PROTOCOL_STATUS_OK;
}
```

这样协议行为会越来越规范。

#### 不要让 Handler 自己乱拼 Frame

又一个架构问题，很容易写成：

```c
static void handle_get_version(...)
{
    uint8_t tx[20];

    tx[0] = 0xAA;
    tx[1] = 0x55;
    ...
    uart_send(tx);
}
```

这样不好，因为 Handler 又开始知道：

```text
Header
Length
CRC
```

协议封装细节再次泄漏。

更好的方式：

```c
protocol_send_response(
    cmd,
    status,
    payload,
    payload_len);
```

由协议层负责：

```text
Header
Length
CMD
Status
Payload
CRC
```

Handler 只负责业务结果。

#### 接收和发送应该对称

接收：

```text
Bytes
↓
Frame Decode
↓
Message
↓
Handler
```

发送：

```text
Handler Result
↓
Message
↓
Frame Encode
↓
Bytes
```

整体：

```text
Receive
                ↓
Bytes → Frame Decoder
             ↓
          Message
             ↓
           Handler
             ↓
          Response
             ↓
Bytes ← Frame Encoder
```

这就是协议层比较完整的形态。

#### 这和“序列化”已经开始靠近

现在我们实际上已经做了：

```text
Frame bytes
→
结构化字段
```

例如：

```text
01 11 22 33
```

解释成：

```c
cmd = 0x01;

payload = {
    0x11,
    0x22,
    0x33
};
```

这其实已经是最简单的：**Deserialize**

发送时反过来：

```text
结构化字段
→
Frame bytes
```

就是：**Serialize**

这正好会为第 6 章铺路。

### 七、一个完整的接收函数

我们可以把前面几课串起来：

```c
parser_result_t protocol_try_parse(
    rx_buffer_t *buffer)
{
    /*
     * 1. 同步 Header
     */
    if (!protocol_sync_to_header(buffer))
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * 2. 固定字段
     */
    if (rx_buffer_size(buffer) < 4)
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * 3. Length
     */
    uint8_t payload_len =
        rx_buffer_at(buffer, FRAME_LEN_OFFSET);

    if (payload_len > MAX_PAYLOAD_SIZE)
    {
        return PARSER_INVALID_LENGTH;
    }

    size_t frame_len =
        5U + payload_len;

    /*
     * 4. 完整 Frame
     */
    if (rx_buffer_size(buffer) < frame_len)
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * 5. CRC
     */
    uint8_t recv_crc =
        rx_buffer_at(buffer, frame_len - 1);

    uint8_t calc_crc =
        protocol_calc_crc(buffer,
                          frame_len - 1);

    if (recv_crc != calc_crc)
    {
        return PARSER_CRC_ERROR;
    }

    /*
     * 6. 提取 Command
     */
    protocol_message_t msg;

    msg.cmd =
        rx_buffer_at(buffer,
                     FRAME_CMD_OFFSET);

    msg.payload_len =
        payload_len;

    /*
     * 假设 Receive Buffer 有连续访问接口
     */
    msg.payload =
        rx_buffer_ptr(buffer,
                      FRAME_PAYLOAD_OFFSET);

    /*
     * 7. 立即交给业务层
     */
    protocol_dispatch(&msg);

    /*
     * 8. Handler 返回后，
     *    Payload 指针不再使用，
     *    可以安全消费 Frame。
     */
    rx_buffer_consume(buffer,
                      frame_len);

    return PARSER_FRAME_OK;
}
```

这就是前面 10 个课时逐步铺出来的结果

**`rx_buffer_ptr()`**** 为什么现在可以出现？**

因为我们采用的是：

```text
滑动线性 Buffer
```

有效数据在内存里始终是连续的，所以可以：

```c
const uint8_t *rx_buffer_ptr(
    const rx_buffer_t *buffer,
    size_t offset)
{
    if (buffer == NULL)
    {
        return NULL;
    }

    if (offset >= buffer->len)
    {
        return NULL;
    }

    return &buffer->data[
        buffer->start + offset
    ];
}
```

于是：

```c
msg.payload =
    rx_buffer_ptr(buffer,
                  FRAME_PAYLOAD_OFFSET);
```

非常方便，这也是上一课我们没有直接上 Ring Buffer 的原因之一。

**如果 Payload 长度为 0 呢？**

完全允许，例如：

```text
GET_VERSION
```

可能没有任何参数。

Frame：

```c
AA 55 00 01 CRC

// LEN = 0 、CMD = 01
```

这时候：

```c
msg.payload_len = 0;
```

Payload 指针可以：

```c
msg.payload = NULL;
```

或者仍然指向某个位置，但业务层绝不能访问。

更推荐：

```c
if (payload_len == 0)
{
    msg.payload = NULL;
}
else
{
    msg.payload =
        rx_buffer_ptr(...);
}
```

语义更明确。

**CMD 也可以设计成 2 Byte**

我们现在为了教学：

```text
CMD = 1 Byte
```

实际上很多设备协议会用：`uint16_t cmd` ，例如：

```text
# 高 Byte = 模块
# 低 Byte = 子命令


0x01xx → 系统管理
0x02xx → 数据操作
0x03xx → 密码算法
```

到时候就必须考虑：

```text
CMD 大小端
```

必须由协议规定。

这再次体现前面学过的：

> 多字节字段一旦跨设备传输，大小端必须明确。
>

**Payload 不能直接强转结构体**

假设 Payload：

```text
11 22 33 44
```

不要轻易这样：

```c
my_struct_t *p =
    (my_struct_t *)msg->payload;

// 然后
value = p->xxx;
```

因为会重新引入：

```text
大小端
Padding
Alignment
编译器布局
```

第 3 章学过的坑全部回来了。

更可靠的是显式解析：

```c
uint16_t value =
    ((uint16_t)msg->payload[0] << 8) |
     (uint16_t)msg->payload[1];
```

这才是真正跨设备稳定的协议，这一点非常重要。

**Protocol Message 和 C Struct 不是一回事**

例如：

```c
typedef struct
{
    uint16_t temperature;
    uint32_t timestamp;

} sensor_data_t;
```

你不能理所当然认为：

```text
wire payload
=
sizeof(sensor_data_t)
```

因为：**内存结构** 和 **传输格式** 是两个概念

正确思路：

```text
Payload 字节
↓
Deserialize
↓
sensor_data_t
```

发送：

```text
sensor_data_t
↓
Serialize
↓
Payload 字节
```

这实际上已经是第 6 章的核心思想了。

### 八、一个推荐的 Command Handler 结构

例如：

```c
static protocol_status_t
handle_set_led(
    const protocol_message_t *msg)
{
    if (msg == NULL)
    {
        return PROTOCOL_STATUS_INTERNAL_ERR;
    }

    if (msg->payload_len != 2)
    {
        return PROTOCOL_STATUS_INVALID_LEN;
    }

    uint8_t led_id = msg->payload[0];
    uint8_t state  = msg->payload[1];

    if (led_id >= LED_COUNT)
    {
        return PROTOCOL_STATUS_INVALID_PARAM;
    }

    if (state > 1)
    {
        return PROTOCOL_STATUS_INVALID_PARAM;
    }

    app_led_set(led_id, state);

    return PROTOCOL_STATUS_OK;
}
```

注意，Handler 完全不知道：

```text
AA 55
CRC
Frame Length
UART
```

这就非常干净。

### 九、Dispatcher 可以进一步升级成表驱动

如果只有几个 Command：

```c
switch
```

完全够，但如果以后：

```text
50 个
100 个
```

Command，可以设计成：

```c
typedef protocol_status_t
(*command_handler_t)(
    const protocol_message_t *msg);
```

表：

```c
typedef struct
{
    uint8_t cmd;
    command_handler_t handler;

} command_entry_t;
```

例如：

```c
static const command_entry_t command_table[] =
{
    { CMD_GET_VERSION, handle_get_version },
    { CMD_SET_LED,     handle_set_led },
    { CMD_GET_STATUS,  handle_get_status }
};
```

Dispatcher 遍历：

```c
for (...)
{
    if (command_table[i].cmd == msg->cmd)
    {
        command_table[i].handler(msg);
        return;
    }
}
```

这就是：**Table\-driven Design，表驱动设计。**

不过本章不再继续展开，否则战线又会拉长。

### 十、这一课要建立的真正思维

以前你看到：

```text
AA 55 03 01 11 22 33 CRC
```

可能会把它看成：一串 Hex。

现在应该分层理解：

```text
Wire Bytes
AA 55 03 01 11 22 33 CRC
         │
         ▼
Frame
Header = AA55
Len    = 3
CMD    = 01
CRC    = ...
         │
         ▼
Message
CMD     = 01
Payload = 11 22 33
         │
         ▼
Business Meaning
执行 Command 0x01
参数 = ...
```

也就是说：**同一串 bytes，在不同层有不同含义。**

这是整个课程“系统中的数据表达与编码”的核心思想之一。

#### 完整的数据接收架构

你现在可以把前面所有内容连起来：

```text
Physical / Transport
                        │
            UART / TCP / USB / RS485
                        │
                        ▼
                   Byte Stream
                        │
                        ▼
                 protocol_feed()
                        │
                        ▼
                Receive Buffer
                        │
                        ▼
                  Frame Parser
          Header / Length / CRC
                        │
                        ▼
                 Valid Frame
                        │
                        ▼
               Message Extraction
                CMD + Payload
                        │
                        ▼
              Command Dispatcher
                        │
                        ▼
                 CMD Handler
                        │
                        ▼
               Application Logic
```

这一整条链，你以后在嵌入式项目里会反复遇到。

---

## 第 5 章 · 第 12 课时

把 Frame、Buffer、Parser、Resync、Dispatcher 组装成一个完整协议模块

不再引入大量新概念，而是做一次真正的“收口”：

```text
protocol.h
protocol.c
rx_buffer
feed
parser
resync
CRC
dispatcher
handler
```

把这一章学过的所有东西串成一份完整 C 代码框架，并最终回答：

> **以后你拿到任何 UART/TCP 二进制私有协议，应该按照什么顺序设计和实现？**
>

先固定协议，我们继续使用第 5 章一直采用的教学协议：

```text
+--------+--------+--------+--------+---------+--------+
| 0xAA   | 0x55   | LEN    | CMD    | Payload | CRC8   |
+--------+--------+--------+--------+---------+--------+
   1B       1B       1B      1B      N Byte     1B
```

规定：

```text
LEN = Payload 长度

# 即：frame_len = 5 + payload_len;
```

```c
#define PROTOCOL_MAX_PAYLOAD_SIZE 64U
```

```c
#define PROTOCOL_RX_BUFFER_SIZE   256U
```

### 一、设计 protocol.h

先不看实现，一个模块设计得好不好，首先看：

> **外部到底需要知道什么？**
>

我们希望 UART、TCP 等传输层只知道：

```c
protocol_init();
protocol_feed(data, len);
```

所以：

```c
#ifndef PROTOCOL_H
#define PROTOCOL_H

#include <stddef.h>
#include <stdint.h>

void protocol_init(void);

void protocol_feed(const uint8_t *data,
                   size_t len);

#endif
```

注意这里故意没有暴露：

```text
rx_buffer
start
len
Parser
CRC
Resync
```

这些全部属于：**protocol\.c 内部实现细节。**

这就是封装。

### 二、逐块看之前的设计

#### （1）Receive Buffer

先建立：

```c
#define PROTOCOL_RX_BUFFER_SIZE 256U

typedef struct
{
    uint8_t data[PROTOCOL_RX_BUFFER_SIZE];

    size_t start;
    size_t len;

} rx_buffer_t;
```

这里继续使用：**滑动线性 Buffer**

有效区域始终是：

```text
data[start]
~
data[start + len - 1]
```

**Buffer 初始化**

```c
static void rx_buffer_init(rx_buffer_t *buffer)
{
    if (buffer == NULL)
    {
        return;
    }

    buffer->start = 0U;
    buffer->len   = 0U;
}
```

清空：

```c
static void rx_buffer_clear(rx_buffer_t *buffer)
{
    if (buffer == NULL)
    {
        return;
    }

    buffer->start = 0U;
    buffer->len   = 0U;
}
```

这里不用：

```c
memset(data, 0, sizeof(data));
```

因为：`len == 0`，就代表没有任何有效数据

**Buffer 有效数据**

```c
static size_t rx_buffer_size(
    const rx_buffer_t *buffer)
{
    if (buffer == NULL)
    {
        return 0U;
    }

    return buffer->len;
}
```

**访问 Buffer 第 i 个逻辑字节**

```c
static uint8_t rx_buffer_at(
    const rx_buffer_t *buffer,
    size_t index)
{
    return buffer->data[
        buffer->start + index
    ];
}
```

注意，这个内部函数默认调用者已经保证：

```c
index < buffer->len
```

在真正要求极严格的库中，也可以额外做边界检查。

**获取连续指针**

因为我们目前不是 Ring Buffer，所以有效数据物理上连续。

```c
static const uint8_t *rx_buffer_ptr(
    const rx_buffer_t *buffer,
    size_t offset)
{
    if (buffer == NULL)
    {
        return NULL;
    }

    if (offset >= buffer->len)
    {
        return NULL;
    }

    return &buffer->data[
        buffer->start + offset
    ];
}
```

**consume**

消费已经处理过的数据：

```c
static int rx_buffer_consume(
    rx_buffer_t *buffer,
    size_t count)
{
    if (buffer == NULL)
    {
        return 0;
    }

    if (count > buffer->len)
    {
        return 0;
    }

    buffer->start += count;
    buffer->len   -= count;

    if (buffer->len == 0U)
    {
        buffer->start = 0U;
    }

    return 1;
}
```

注意这里：**没有 memmove，只是 start 往后移动**

**compact**

只有尾部连续空间不足时，才整理 Buffer。

```c
static void rx_buffer_compact(
    rx_buffer_t *buffer)
{
    if (buffer == NULL)
    {
        return;
    }

    if (buffer->start == 0U)
    {
        return;
    }

    if (buffer->len > 0U)
    {
        memmove(buffer->data,
                buffer->data + buffer->start,
                buffer->len);
    }

    buffer->start = 0U;
}
```

**append**

```c
static int rx_buffer_append(
    rx_buffer_t *buffer,
    const uint8_t *data,
    size_t len)
{
    size_t end;
    size_t tail_space;

    if (buffer == NULL)
    {
        return 0;
    }

    if (len == 0U)
    {
        return 1;
    }

    if (data == NULL)
    {
        return 0;
    }

    /*
     * 总容量不足
     */
    if (buffer->len + len >
        PROTOCOL_RX_BUFFER_SIZE)
    {
        return 0;
    }

    end =
        buffer->start +
        buffer->len;

    tail_space =
        PROTOCOL_RX_BUFFER_SIZE -
        end;

    /*
     * 总空间够，但尾部连续空间不足
     */
    if (tail_space < len)
    {
        rx_buffer_compact(buffer);

        end = buffer->len;
    }

    memcpy(buffer->data + end,
           data,
           len);

    buffer->len += len;

    return 1;
}
```

到这里：

```text
Receive Buffer
```

模块已经基本完成。

#### （2）定义协议量

```c
#define PROTOCOL_HEADER0             0xAAU
#define PROTOCOL_HEADER1             0x55U

#define PROTOCOL_MAX_PAYLOAD_SIZE    64U

#define PROTOCOL_LEN_OFFSET          2U
#define PROTOCOL_CMD_OFFSET          3U
#define PROTOCOL_PAYLOAD_OFFSET      4U

#define PROTOCOL_FIXED_SIZE          5U
// 这里：FIXED_SIZE = Header + LEN + CMD + CRC = 2 + 1 + 1 + 1 = 5
```

于是：

```c
frame_len =
    PROTOCOL_FIXED_SIZE +
    payload_len;
```

比直接写：`5 + payload_len` 更清晰。

**定义 Parser 返回值**

```c
typedef enum
{
    PARSER_NEED_MORE_DATA = 0,
    PARSER_FRAME_OK,
    PARSER_INVALID_LENGTH,
    PARSER_CRC_ERROR

} parser_result_t;
```

我们已经知道：

```text
FRAME_OK
→ 消费整帧

ERROR
→ 至少前进 1 Byte

NEED_MORE
→ 保留数据并退出
```

**Message**

Parser 不直接把 Frame 暴露给业务层，定义：

```c
typedef struct
{
    uint8_t cmd;

    const uint8_t *payload;
    size_t payload_len;

} protocol_message_t;
```

业务层只关心：

```text
CMD
Payload
```

而不是：

```text
Header
CRC
Frame Length
```

**Command**

假设演示三个命令：

```c
#define CMD_GET_VERSION  0x01U
#define CMD_SET_LED      0x02U
#define CMD_GET_STATUS   0x03U
```

实际项目当然可以替换成自己的命令体系。

### 三、同步到 Header

这是 Resync 核心函数。

```c
static int protocol_sync_to_header(
    rx_buffer_t *buffer)
{
    size_t len;
    size_t i;

    len = rx_buffer_size(buffer);

    if (len == 0U)
    {
        return 0;
    }

    for (i = 0U; i + 1U < len; i++)
    {
        if ((rx_buffer_at(buffer, i)
                == PROTOCOL_HEADER0) &&
            (rx_buffer_at(buffer, i + 1U)
                == PROTOCOL_HEADER1))
        {
            /*
             * 丢掉 Header 前面的垃圾
             */
            if (i > 0U)
            {
                rx_buffer_consume(buffer, i);
            }

            return 1;
        }
    }

    /*
     * 没找到完整 AA 55。
     *
     * 但最后一个字节如果是 AA，
     * 必须保留，因为下一批数据
     * 可能以 55 开头。
     */
    if (rx_buffer_at(buffer, len - 1U)
        == PROTOCOL_HEADER0)
    {
        rx_buffer_consume(
            buffer,
            len - 1U);
    }
    else
    {
        rx_buffer_consume(
            buffer,
            len);
    }

    return 0;
}
```

这段代码需要真正理解，因为它解决：

```text
垃圾 + Header
```

以及：

```text
Header 被拆成两次接收
```

两种问题。

### 四、CRC 函数

CRC 具体算法不是这一课重点，我们先定义一个接口：

```c
static uint8_t protocol_crc8(
    const uint8_t *data,
    size_t len)
{
    uint8_t crc = 0U;
    size_t i;

    for (i = 0U; i < len; i++)
    {
        crc ^= data[i];
    }

    return crc;
}
```

注意：这里其实只是为了教学使用的简单 XOR 校验示例。

**真正工程里应该换成协议明确规定的：**

```text
CRC-8
CRC-16
CRC-32
Checksum
```

并明确：

```text
Polynomial
Init
RefIn
RefOut
XorOut
参与 CRC 的字段范围
```

不能看到“CRC8”三个字就自己猜算法。

### 五、Dispatcher

先写 Handler，例如：

```c
static void handle_get_version(
    const protocol_message_t *msg)
{
    if (msg->payload_len != 0U)
    {
        return;
    }

    /*
     * Application logic
     *
     * 例如返回软件版本
     */
}
```

SET\_LED：

```c
static void handle_set_led(
    const protocol_message_t *msg)
{
    uint8_t led_id;
    uint8_t state;

    if (msg->payload_len != 2U)
    {
        return;
    }

    led_id = msg->payload[0];
    state  = msg->payload[1];

    /*
     * 参数检查
     */
    if (state > 1U)
    {
        return;
    }

    /*
     * app_led_set(led_id, state);
     */
}
```

状态查询：

```c
static void handle_get_status(
    const protocol_message_t *msg)
{
    if (msg->payload_len != 0U)
    {
        return;
    }

    /*
     * Application logic
     */
}
```

Dispatcher：

```c
static void protocol_dispatch(
    const protocol_message_t *msg)
{
    if (msg == NULL)
    {
        return;
    }

    switch (msg->cmd)
    {
    case CMD_GET_VERSION:
        handle_get_version(msg);
        break;

    case CMD_SET_LED:
        handle_set_led(msg);
        break;

    case CMD_GET_STATUS:
        handle_get_status(msg);
        break;

    default:
        /*
         * Unsupported Command
         */
        break;
    }
}
```

### 六、进入核心：try_parse()

这就是整个第 5 章的核心函数。

```c
static parser_result_t
protocol_try_parse(rx_buffer_t *buffer)
{
    uint8_t payload_len;
    size_t frame_len;

    const uint8_t *frame;

    uint8_t received_crc;
    uint8_t calculated_crc;

    protocol_message_t msg;
```

#### 第一步：找 Header

```c
if (!protocol_sync_to_header(buffer))
    {
        return PARSER_NEED_MORE_DATA;
    }
```

这一步执行完成以后：`buffer[0]` 逻辑上一定是 **AA** ，**buffer\[1\] = 55**

#### 第二步：检查固定字段

至少需要：

```text
AA 55 LEN CMD
```

所以：

```c
if (rx_buffer_size(buffer) < 4U)
    {
        return PARSER_NEED_MORE_DATA;
    }
```

注意：我们还不要求 CRC 已经收到，此时只是为了读取 **LEN**

#### 第三步：读 Length

```c
payload_len =
        rx_buffer_at(
            buffer,
            PROTOCOL_LEN_OFFSET);
```

然后必须检查：

```c
if (payload_len >
        PROTOCOL_MAX_PAYLOAD_SIZE)
    {
        return PARSER_INVALID_LENGTH;
    }
```

绝不能无限相信对端的 Length。

#### 第四步：计算完整 Frame 长度

```c
frame_len =
        PROTOCOL_FIXED_SIZE +
        payload_len;
```

例如：**LEN = 3** ，得到 **Frame Len = 5 \+ 3 = 8**

#### 第五步：检查数据是否到齐

```c
if (rx_buffer_size(buffer) <
        frame_len)
    {
        return PARSER_NEED_MORE_DATA;
    }
```

如果这里数据不够：**什么都不要删。**

等待下一次：

```c
protocol_feed();
```

继续追加。

#### 第六步：获取 Frame 指针

```c
frame =
        rx_buffer_ptr(buffer, 0U);

    if (frame == NULL)
    {
        return PARSER_NEED_MORE_DATA;
    }
```

此时整个 Frame 已经连续存在内存中。

#### 第七步：CRC 校验

我们的示例规定：

```text
最后一个 Byte = CRC
```

所以：

```c
received_crc =
        frame[frame_len - 1U];
```

CRC 计算范围假设为：

```text
Header
+
LEN
+
CMD
+
Payload
```

不包含 CRC 自己：

```c
calculated_crc =
        protocol_crc8(
            frame,
            frame_len - 1U);
```

判断：

```c
if (received_crc != calculated_crc)
    {
        return PARSER_CRC_ERROR;
    }
```

#### 第八步：构造 Message

现在 Frame 已经：

```text
完整
+
CRC 正确
```

可以放心提取：

```c
msg.cmd =
        frame[PROTOCOL_CMD_OFFSET];

    msg.payload_len =
        payload_len;
```

Payload：

```c
if (payload_len == 0U)
    {
        msg.payload = NULL;
    }
    else
    {
        msg.payload =
            frame +
            PROTOCOL_PAYLOAD_OFFSET;
    }
```

#### 第九步：Dispatch

```c
protocol_dispatch(&msg);
```

这里采用：**同步 \+ Zero\-copy **

所以 **msg\.payload**** **直接指向 **Receive Buffer**

因此必须在：

```c
rx_buffer_consume()
```

之前处理完。

#### 第十步：消费整个合法 Frame

```c
rx_buffer_consume(
        buffer,
        frame_len);
```

然后：

```c
return PARSER_FRAME_OK;
}
```

#### 完整函数：

```c
static parser_result_t
protocol_try_parse(
    rx_buffer_t *buffer)
{
    uint8_t payload_len;
    size_t frame_len;

    const uint8_t *frame;

    uint8_t received_crc;
    uint8_t calculated_crc;

    protocol_message_t msg;

    /*
     * 1. Header / Resync
     */
    if (!protocol_sync_to_header(buffer))
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * 2. Header + LEN + CMD
     */
    if (rx_buffer_size(buffer) < 4U)
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * 3. Length
     */
    payload_len =
        rx_buffer_at(
            buffer,
            PROTOCOL_LEN_OFFSET);

    if (payload_len >
        PROTOCOL_MAX_PAYLOAD_SIZE)
    {
        return PARSER_INVALID_LENGTH;
    }

    /*
     * 4. Full frame size
     */
    frame_len =
        PROTOCOL_FIXED_SIZE +
        payload_len;

    /*
     * 5. Complete?
     */
    if (rx_buffer_size(buffer) <
        frame_len)
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * 6. Frame pointer
     */
    frame =
        rx_buffer_ptr(buffer, 0U);

    if (frame == NULL)
    {
        return PARSER_NEED_MORE_DATA;
    }

    /*
     * 7. CRC
     */
    received_crc =
        frame[frame_len - 1U];

    calculated_crc =
        protocol_crc8(
            frame,
            frame_len - 1U);

    if (received_crc != calculated_crc)
    {
        return PARSER_CRC_ERROR;
    }

    /*
     * 8. Frame -> Message
     */
    msg.cmd =
        frame[PROTOCOL_CMD_OFFSET];

    msg.payload_len =
        payload_len;

    if (payload_len == 0U)
    {
        msg.payload = NULL;
    }
    else
    {
        msg.payload =
            frame +
            PROTOCOL_PAYLOAD_OFFSET;
    }

    /*
     * 9. Message -> Application
     */
    protocol_dispatch(&msg);

    /*
     * 10. Consume valid frame
     */
    rx_buffer_consume(
        buffer,
        frame_len);

    return PARSER_FRAME_OK;
}
```

到这里，你已经拥有一个真正完整的 Parser 核心。

### 七、最后缺的就是 protocol_feed()

先定义全局 Buffer：

```c
static rx_buffer_t g_rx_buffer;
```

初始化：

```c
void protocol_init(void)
{
    rx_buffer_init(&g_rx_buffer);
}
```

#### feed 的第一步：append

```c
void protocol_feed(
    const uint8_t *data,
    size_t len)
{
    parser_result_t result;

    if ((data == NULL) && (len > 0U))
    {
        return;
    }

    if (!rx_buffer_append(
            &g_rx_buffer,
            data,
            len))
    {
        /*
         * Buffer Overflow
         *
         * 当前策略：
         * 清空重新同步
         */
        rx_buffer_clear(&g_rx_buffer);

        return;
    }
```

#### 第二步：不停 Parse

```c
while (1)
    {
        result =
            protocol_try_parse(
                &g_rx_buffer);
```

针对三类结果：

**（1）FRAME\_OK**

```c
if (result ==
            PARSER_FRAME_OK)
        {
            continue;
        }
```

因为：**Buffer 后面可能还有 Frame**

**（2）NEED\_MORE\_DATA**

```c
if (result ==
            PARSER_NEED_MORE_DATA)
        {
            return;
        }
```

这是唯一正常退出解析循环的情况。

**（3）ERROR**

```c
if ((result ==
             PARSER_INVALID_LENGTH) ||
            (result ==
             PARSER_CRC_ERROR))
        {
            if (rx_buffer_size(
                    &g_rx_buffer) > 0U)
            {
                rx_buffer_consume(
                    &g_rx_buffer,
                    1U);
            }

            continue;
        }
```

记住：

```text
错误
→ consume(1)
→ 重新找 Header
```

#### 完整 feed()

```c
void protocol_feed(
    const uint8_t *data,
    size_t len)
{
    parser_result_t result;

    if ((data == NULL) && (len > 0U))
    {
        return;
    }

    /*
     * 新字节进入 Receive Buffer
     */
    if (!rx_buffer_append(
            &g_rx_buffer,
            data,
            len))
    {
        /*
         * Buffer Overflow
         */
        rx_buffer_clear(
            &g_rx_buffer);

        return;
    }

    while (1)
    {
        result =
            protocol_try_parse(
                &g_rx_buffer);

        switch (result)
        {
        case PARSER_FRAME_OK:

            /*
             * 可能还有下一帧
             */
            continue;

        case PARSER_NEED_MORE_DATA:

            /*
             * 当前数据不够，
             * 等下一批 bytes。
             */
            return;

        case PARSER_INVALID_LENGTH:
        case PARSER_CRC_ERROR:

            /*
             * 当前候选 Header 不可信。
             *
             * 最小丢弃 1 Byte，
             * 然后重新同步。
             */
            if (rx_buffer_size(
                    &g_rx_buffer) > 0U)
            {
                rx_buffer_consume(
                    &g_rx_buffer,
                    1U);
            }

            continue;

        default:
            return;
        }
    }
}
```

这就是这一章最终需要掌握的主函数。

### 八、UART 接入

例如：

```c
void uart_rx_callback(
    const uint8_t *data,
    size_t len)
{
    protocol_feed(data, len);
}
```

**底层一次收到：**AA 55 03** ，**没关系。

*Parser：*

```text
NEED_MORE
```

**第二次：***01 11 22*

还是可能：

```text
NEED_MORE
```

**第三次：***33 CRC*

拼成完整 *Frame*，*Parser* 自动处理。

### 九、TCP 接入

完全一样：

```c
uint8_t buf[128];

int n = recv(
    sock,
    buf,
    sizeof(buf),
    0);

if (n > 0)
{
    protocol_feed(
        buf,
        (size_t)n);
}
```

注意这里非常关键：

> Parser 完全不知道这是 UART 还是 TCP。
>

它只知道：**bytes**

这就是正确分层带来的好处。

### 十、拆包测试

假设完整 Frame：

```text
AA 55 03 01 11 22 33 CRC
```

底层分成：

```text
# 第 1 次：
AA

# 第 2 次：
55 03

# 第 3 次：
01 11

# 第 4 次：
22 33 CRC
```

每一次都调用：

```c
protocol_feed();
```

最终：**Receive Buffer** ，会自动拼成完整 Frame。

这就是：**解决拆包。**

### 十一、粘包测试

一次收到：

```text
Frame1 | Frame2 | Frame3
```

例如：

```text
AA 55 00 01 CRC
AA 55 02 02 01 01 CRC
AA 55 00 03 CRC
```

`protocol_feed()` 只调用一次，但是内部：

```c
while (1)
```

会：

```text
解析 Frame1
consume Frame1
↓
继续

解析 Frame2
consume Frame2
↓
继续

解析 Frame3
consume Frame3
↓
继续

Buffer 空
↓
NEED_MORE
↓
退出
```

这就是：**解决粘包。**

### 十三、噪声测试

收到：

```text
91 27 FF 13
AA 55 00 01 CRC
```

`protocol_sync_to_header()` 会：

```text
丢：

91 27 FF 13
```

保留：

```text
AA 55 00 01 CRC
```

然后正常解析。

这就是：**帧同步。**

### 十四、错误 Length 测试

收到：

```text
AA 55 FF ...
```

但：**MAX\_PAYLOAD = 64**

于是：

```text
INVALID_LENGTH
```

然后：**consume\(1\);**

重新寻找：

```text
AA 55
```

而不是傻等：**260 Byte**

### 十五、CRC 错误测试

收到：

```text
AA 55 03 01 11 22 33 错误CRC
AA 55 00 03 正确CRC
```

第一帧：

```text
CRC_ERROR
```

不会：**clear buffer**

而是：

```text
consume(1)
↓
Resync
```

于是仍然有机会找到后面那一帧。

### 十六、整个 Parser 为什么不会死循环？

回忆第 10 课时的一条核心规则，每一轮：

```text
要么消费数据
要么退出
```

三种情况：

**FRAME\_OK**

```text
consume(frame_len)
```

Buffer 前进。

**ERROR**

```text
consume(1)
```

Buffer 前进。

**NEED\_MORE**

```text
return
```

退出。

所以永远不会出现：

```text
同一批数据
反复 parse
但什么都不变
```

这就是 Parser 的：**前进性保证。**

### 十七、真正项目里建议加 Stats

例如：

```c
typedef struct
{
    uint32_t frames_ok;

    uint32_t crc_errors;
    uint32_t length_errors;

    uint32_t overflow_count;
    uint32_t resync_count;

} protocol_stats_t;
```

然后：

```text
FRAME_OK
→ frames_ok++

CRC_ERROR
→ crc_errors++

INVALID_LENGTH
→ length_errors++

Overflow
→ overflow_count++
```

现场排查非常有价值，不过这属于：

```text
增强项
```

而不是 Parser 的核心必要条件。

### 十八、真正工程还要考虑 ISR 问题

例如 UART 中断里收到 Byte：

```c
void UART_IRQHandler(void)
{
    ...
}
```

不要默认就在 ISR 中：

```c
protocol_feed();
```

更不要直接：

```text
解析 Frame
处理 Flash
执行密码算法
发复杂回复
```

因为 ISR 应尽量短。

**常见架构是：**

```text
UART ISR / DMA
      ↓
Ring Buffer / Queue
      ↓
Protocol Task
      ↓
protocol_feed()
```

**或者裸机：**

```text
ISR 收数据
↓
放 Buffer
↓
main loop 中调用 Parser
```

这和我们这章的 Frame Parser 并不冲突，只是：

> Parser 应该运行在哪个上下文，是另一个系统设计问题。
>

### 十九、这一章理解

#### （1）为什么一直强调“不要把 read 当 Frame”？

现在你已经可以完整回答了，因为：

```c
read()
recv()
UART DMA callback
```

它们解决的是：**数据什么时候从硬件/内核来到软件。**

而 Frame Parser 解决的是：

> **这些 Byte 在协议语义上从哪里开始，到哪里结束。**
>

这两个边界没有必然关系，例如：

```text
一次 recv：

半个 Frame
```

完全正常，一次 recv：

```text
3.5 个 Frame
```

也完全正常。

所以：**传输边界 ≠ 协议帧边界****。 **这是第 5 章最重要的结论之一。

#### （2）以后设计一个私有协议，正确顺序

以后领导给你一个需求：

> **“MCU 和上位机通过串口通信，自己定义个协议。”**
>

不要立刻写代码，按照下面这个顺序。

**第一步：定义 Wire Format**

例如：

```text
Header
Version
Length
Sequence
Command
Status
Payload
CRC
```

明确每个字段：

```text
长度
含义
大小端
取值范围
```

**第二步：明确 Length 定义**

是：

- **Payload Length**
- **整个 Frame Length**
- **从 CMD 到 CRC 的长度**

究竟是什么，必须写进协议文档。

**第三步：定义最大 Frame**

例如：

```text
MAX_PAYLOAD = 1024
```

那么必须计算：**MAX\_FRAME\_SIZE**

用来设计：**Buffer Size**

**第四步：明确 CRC**

必须说明：

```text
算法
参数
字节序
计算范围
```

而不是简单写：

```text
CRC16
```

**第五步：先写 Encode / Decode 规则**

不要：

```c
send(struct)
```

而是明确：

```text
整数怎么转 Byte
Byte 怎么恢复整数
```

**第六步：设计 Stream Parser**

必须考虑：

```text
拆包
粘包
噪声
错误 Length
CRC Error
Resync
```

**第七步：协议层与业务层分离**

```text
Frame Parser
↓
Message
↓
Dispatcher
↓
Application
```

不要混在一起。

### 二十、第 5 章的总图

到这里，这张图你应该能完全理解：

```text
Continuous Byte Stream
                          │
                          ▼
                ┌───────────────────┐
                │   Receive Buffer  │
                │ append / consume  │
                └─────────┬─────────┘
                          │
                          ▼
                  Search Sync Word
                     AA 55
                          │
                          ▼
                  Enough fixed bytes?
                    │           │
                   No          Yes
                    │           │
                NEED_MORE       ▼
                            Read Length
                                 │
                                 ▼
                          Length valid?
                           │         │
                          No        Yes
                           │         │
                     consume(1)      ▼
                        Resync   Full frame?
                                  │       │
                                 No      Yes
                                  │       │
                              NEED_MORE   ▼
                                         CRC
                                      │       │
                                     Fail     OK
                                      │       │
                                consume(1)    ▼
                                   Resync   Message
                                             │
                                             ▼
                                          CMD +
                                         Payload
                                             │
                                             ▼
                                        Dispatcher
                                             │
                                             ▼
                                        Application
                                             │
                                             ▼
                                   consume(frame_len)
                                             │
                                             ▼
                                      Continue Parse
```

这就是第 5 章的最终答案。

**现在应该能区分这些概念了**

**Byte**

> ```text
> 0xAA
> ```
>
> 只是一个 8\-bit 数值。
>

**Byte Stream**

> ```text
> AA 55 03 01 ...
> ```
>
> 只是连续传输的字节。
>

**Frame**

> ```text
> Header + Length + CMD + Payload + CRC
> ```
>
> 是协议层定义出来的边界。
>

**Message**

> ```text
> CMD + Payload
> ```
>
> 是协议层提取出来的语义数据。
>

**Command**

> ```text
> CMD = 0x02
> ```
>
> 是业务层理解的操作。
>

**这些概念不要再混在一起。**

> 为什么理解各种数据格式、编码和协议，对于嵌入式不同设备之间交流很重要？
>

现在答案已经非常具体了，假设你抓到：

```text
AA 55 02 01 12 34 9F
```

你不能只知道：*这是 Hex*

你还需要知道：

```text
Hex
↓
表示 Bytes

Bytes
↓
属于 Byte Stream

Byte Stream
↓
根据协议切成 Frame

Frame
↓
按字段解释

Payload
↓
按照规定的编码、大小端、序列化规则继续解释
```

数据不是“天然有意义”的。

**意义是协议逐层赋予它的****。 **这正是这门课程真正要建立的能力。

第 5 章我们最终完成了：

```text
Frame
Length
Command
Payload
CRC
Byte Stream
粘包 / 拆包
Receive Buffer
流式 Parser
Resync
Dispatcher
协议层 / 业务层分离
```

到这里，第 5 章可以正式封章。
