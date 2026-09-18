# 系统中的数据表达、编码与传输：从 Bite 到跨设备协议 六章

## 第 6 章 · 第 1 课时

**Serialization：序列化到底是什么**

这一课我们先不碰 ASN.1、DER，先把最根本的问题讲清楚：

> **程序内存里的“数据结构”，为什么不能直接拿去传输？**
>

以及：

> **序列化到底是在做什么？**
>

这两个问题一旦彻底明白，后面的 TLV、ASN.1、DER、Protobuf、JSON，都会变得非常自然。

### 一、序列化核心思想

假设 MCU 里有这样一个结构体：

```c
typedef struct
{
    uint8_t  cmd;
    uint16_t length;
    uint32_t value;
} Message;
```

然后：

```c
Message msg;

msg.cmd    = 0x01;
msg.length = 0x0004;
msg.value  = 0x12345678;
```

我们第一反应就是：

```c
uart_send((uint8_t *)&msg, sizeof(msg));
```

也就是：**直接把 struct 当成一串 byte 发出去。**

这看起来非常合理，但我们通过前几章的学习知道这其实隐藏着很多问题。

**程序看到的是“字段”，链路看到的只有 Byte**

程序员眼中的：

```c
Message msg;

有
    cmd
    length
    value
```

但 UART、TCP、SPI、USB 等链路根本不知道：

```text
cmd
length
value
```

链路只能看到：

```text
01 00 04 00 78 56 34 12
# 或者
01 04 00 78 56 34 12
# 或者
01 00 04 12 34 56 78
# 甚至别的形式
```

也就是对于链路来说，世界里只有：

```text
Byte Byte Byte Byte Byte ...
```

所以：**“这个 Byte 代表什么”，必须由协议规定。**

这已经是序列化的核心思想了。

### 二、什么叫 Serialization

Serialization 中文通常叫：**序列化**

它本质上就是：

> 把程序内部的数据对象，按照某一种确定的规则，
>
> 转换成一串可以保存或传输的 Byte Sequence。
>

例如内存里的：

```text
cmd    = 1
length = 4
value  = 0x12345678
```

经过某种规则，变成：

```text
01 00 04 12 34 56 78
```

这就是序列化，整个过程：

```text
程序中的数据
       │
       ▼
Serialization
       │
       ▼
Byte Sequence
       │
       ▼
UART / TCP / Flash / File
```

反过来：

```text
Byte Sequence
       │
       ▼
Deserialization
       │
       ▼
程序中的数据
```

就是：**反序列化（Deserialization）**

### 三、为什么不直接发送 struct

这是这一课最重要的部分。

我们在第 3 章其实已经学过一些问题，现在正好全部串起来，还是：

```c
typedef struct
{
    uint8_t  cmd;
    uint16_t length;
    uint32_t value;
} Message;
```

你可能觉得它应该占：

```text
1 + 2 + 4 = 7 bytes
```

但实际上：`sizeof(Message)` ，很可能是 **8**

因为：

```text
Padding
Alignment
```

**1. Padding 会混进传输数据**

一种可能的内存布局：

```text
Offset
0       cmd
1       padding
2~3     length
4~7     value

于是：sizeof(Message) = 8
```

真实内存：

```text
01 ?? 04 00 78 56 34 12
```

其中：`??` 就是 padding。

它根本不是你的业务数据，但：

```c
uart_send((uint8_t *)&msg, sizeof(msg));
```

会把它一起发出去。

**Padding 还不是最严重的问题，**真正麻烦的是：

> **不同编译器、不同 CPU、不同 ABI，struct 的内存布局不一定完全一致。**
>

例如：

```text
# 设备 A：
cmd
padding
length
value

# 设备 B：
cmd
length
value
```

甚至因为：

```c
#pragma pack(1)
```

可能又变成另一种布局。

也就是说：

```text
C struct
```

本质上属于：**程序的内存实现**

而协议需要的是：

> **设备之间共同约定的外部数据格式**
>

这两个概念一定要分开。

**2.*** ***大小端问题**

这个已经很熟了，假设：

```c
uint32_t value = 0x12345678;
```

在小端 MCU 内存里可能是：

```text
78 56 34 12
```

但如果协议规定：

```text
Network Byte Order
Big Endian
```

那么线上必须是：

```text
12 34 56 78
```

所以：**内存布局** 并不等于 **协议布局**

### 四、两个重要的词

从现在开始建议你牢牢记住：

```text
Memory Representation
```

和：

```text
Wire Format
```

**Memory Representation**

就是：**数据在程序运行时，在 RAM 里的样子。**例如

```text
78 56 34 12
```

可能只是：

```c
uint32_t value = 0x12345678;
```

在小端 CPU 中的内存形式。

**Wire Format**

Wire Format 可以理解为：

> 数据在通信线路上规定必须采用的格式。
>

例如协议明确规定：

```text
Byte 0      Command
Byte 1~2    Length，Big Endian
Byte 3~6    Value，Big Endian
```

那么：

```text
01 00 04 12 34 56 78
```

才是真正的 Wire Format。

**注意：**

```text
Wire
```

这里并不一定真的意味着“电线”，它泛指：

```text
UART
TCP
USB
SPI
文件
Flash
共享数据
```

等外部交换形式。

### 五、序列化实际上是在做一次“翻译”

可以这样理解：

```text
# C 程序世界的数据：
msg.cmd
msg.length
msg.value
```

**经过：Serialization 翻译为**

```text
# Protocol World 里面
01 00 04 12 34 56 78
```

所以：

```text
struct 是：
        程序表达

而 01 00 04 12 34 56 78 是：
        协议外部表达
```

这就是第 6 章真正开始进入的领域：**数据表达的“内外转换”**

（1）自己设计一个最简单的序列化格式，假设协议规定：

```text
# 字段        大小        编码规则
 cmd        1 Byte        原值
 length     2 Byte        Big Endian
 valie      4 Byte        Big Endian
```

那么：

```c
cmd    = 0x01;
length = 0x0004;
value  = 0x12345678;
```

序列化结果：

```text
01
00 04
12 34 56 78
```

合起来：

```text
01 00 04 12 34 56 78
```

注意这里固定是：**7 bytes**

不管：**sizeof(Message) 是多少**

这就是一个非常重要的原则：

> **协议大小不应该依赖 sizeof(struct)。**
>

（2）在 C 里面应该怎么做？

工业代码一般会明确写入 buffer，例如：

```c
uint8_t buf[7];

buf[0] = msg.cmd;

buf[1] = (uint8_t)(msg.length >> 8);
buf[2] = (uint8_t)(msg.length);

buf[3] = (uint8_t)(msg.value >> 24);
buf[4] = (uint8_t)(msg.value >> 16);
buf[5] = (uint8_t)(msg.value >> 8);
buf[6] = (uint8_t)(msg.value);
```

最后：

```text
buf =
01 00 04 12 34 56 78
```

这个操作：

```text
Message
    ↓
uint8_t buffer[]
```

就是：**Serialization**

（3）接收端反过来，接收端收到：

```text
01 00 04 12 34 56 78
```

然后：

```c
Message msg;

msg.cmd = buf[0];

msg.length =
    ((uint16_t)buf[1] << 8) |
    ((uint16_t)buf[2]);

msg.value =
    ((uint32_t)buf[3] << 24) |
    ((uint32_t)buf[4] << 16) |
    ((uint32_t)buf[5] << 8)  |
    ((uint32_t)buf[6]);
```

于是恢复成：

```text
cmd    = 0x01
length = 0x0004
value  = 0x12345678
```

这就是：**Deserialization**

也叫：

```text
Decode
Parse
Unmarshal
Deserialize
# 具体项目术语可能不同。
```

### 六、与第 5 章关系

第 5 章我们学的是：

```text
Frame
```

例如：

```text
AA 55 | Length | Command | Payload | CRC
```

现在来看：

```text
Payload
```

其实也不能只是模糊地说：**“这里放业务数据。”**

因为 Payload 里很可能还有：

```text
Device ID
Timestamp
Status
Key
Algorithm
Data
```

那这些字段怎么排列？比如：

```text
Payload
│
├─ Device ID
├─ Timestamp
├─ Status
└─ Data
```

仍然需要一种：**Serialization Rule**

所以：

```text
第 5 章
解决 Frame 边界
```

而：

```text
第 6 章
解决 Frame 内部的数据组织
```

完整关系是：

```text
Byte Stream
    │
    ▼
Frame Parser
    │
    ▼
完整 Frame
    │
    ├─ Header
    ├─ Length
    ├─ Command
    │
    ├─ Payload
    │     │
    │     ▼
    │ Serialization Format
    │     │
    │     ├─ Field A
    │     ├─ Field B
    │     └─ Field C
    │
    └─ CRC
```

现在前五章和第六章就真正连起来了。

### 七、最简单的序列化方式：Fixed Layout

刚才我们设计的：

```text
Byte 0       cmd
Byte 1~2     length
Byte 3~6     value
```

这种方式就叫：**Fixed Layout**

可以翻译成：**固定布局编码**

协议事先明确规定：

```text
哪个字段
在什么 offset
占多少 byte
什么 endian
```

例如：

```text
Offset    Size    Field
0         1       cmd
1         2       length
3         4       value
```

于是接收端无需任何字段说明，看到：

```text
01 00 04 12 34 56 78
```

它直接知道：

```text
byte 0   = cmd
byte 1~2 = length
byte 3~6 = value
```

Fixed Layout 在嵌入式里非常常见

比如 MCU 协议：

```text
Byte 0     Command
Byte 1     Status
Byte 2~3   Length
Byte 4~7   Device ID
Byte 8...  Data
```

这种做法优点非常明显：

```text
简单
快
占空间少
MCU 很容易解析
```

所以你会在很多：

```text
MCU ↔ MCU
MCU ↔ Linux
芯片驱动
Bootloader
工业设备
自定义串口协议
```

里面看到这种格式。

但 Fixed Layout 有一个明显问题

假设第一版协议：

```text
Byte 0    cmd
Byte 1    status
Byte 2~5  value
```

后来产品经理说：**增加一个 device_type。**

你可能改成：

```text
Byte 0    cmd
Byte 1    status
Byte 2    device_type
Byte 3~6  value
```

问题来了，老版本程序还认为：

```text
Byte 2~5
是
value
```

于是协议直接错位。

这就是 Fixed Layout 的弱点：**字段位置耦合比较严重。**

这就引出了下一课的 TLV

如果我们不是规定：

```text
Byte 0 是 cmd
Byte 1 是 status
```

而是每个字段自己带上：

```text
我是谁
我多长
我的数据是什么
```

就变成：

```text
Tag
Length
Value
```

也就是：**TLV**

例如：

```text
01 01 05
```

可以理解为：

```text
Tag    = 01
Length = 01
Value  = 05
```

接收端不再完全依赖：

```text
offset
```

而可以依据：**Tag，**来识别字段。

这就是为什么下一课我们会专门研究 TLV。

### 八、序列化不是“转成字符串”

这是一个很容易产生的误解，很多人第一次接触序列化，会觉得：

> 序列化是不是就是变成 JSON？
>

JSON 只是序列化的一种方式。

例如对象：

```text
cmd = 1
value = 100
```

可以序列化成 JSON：

```json
{
  "cmd": 1,
  "value": 100
}
```

也可以序列化成：

```text
01 00 00 00 64
```

也可以变成 TLV：

```text
01 01 01
02 04 00 00 00 64
```

甚至可以变成 ASN.1 DER。

所以：**序列化描述的是“过程”，不是某一种具体格式。**

### 九、重要的思想

你以后会大量见到这些词：

```text
Serialization         把程序数据转成传输/保存格式
Deserialization       把外部格式恢复为程序数据
Encoding              按某套规则编码
Decoding**              按规则恢复**
Parsing**               分析 byte 结构并提取字段**
Wire Format**           数据在通信/存储层面的实际格式**
Schema**                描述有哪些字段、字段类型及结构**
```

这里不要死抠每个英文词绝对边界。

实际工程里：

```text
encode()
serialize()
marshal()
```

经常有部分语义重叠。

而：

```text
decode()
deserialize()
unmarshal()
parse()
```

也经常重叠。

真正需要理解的是数据流：

```text
程序内部结构
      ↓
按照协议转换
      ↓
Byte Sequence
      ↓
传输
      ↓
Byte Sequence
      ↓
解析
      ↓
程序内部结构
```

假设你的 MCU 中：

```c
typedef struct
{
    uint32_t id;
    uint16_t status;
} DeviceInfo;
```

最好不要认为：

```c
DeviceInfo
```

本身就是协议。

应该明确区分成：

```text
DeviceInfo
=
内部数据模型
```

而：

```text
device_info_encode()
#      负责
# 内部模型 → Wire Format

device_info_decode()
#      负责
# Wire Format → 内部模型
```

例如：

```c
int device_info_encode(
    const DeviceInfo *info,
    uint8_t *buf,
    size_t buf_size);

int device_info_decode(
    const uint8_t *buf,
    size_t len,
    DeviceInfo *info);
```

这种设计比：

```c
send((uint8_t *)&info, sizeof(info));
```

健壮得多。

这里其实有一道分界线

这是我希望你这一课真正建立起来的思维，以前你可能认为：

```text
uint32_t value = 0x12345678;
```

这个数据就是：

```text
78 56 34 12
```

现在应该知道：不是，准确来说：

```text
0x12345678
```

是 **数值****，**而：

```text
78 56 34 12
```

只是它在某个小端 CPU 中的一种：**Memory Representation**

协议完全可以规定它必须发送为：

```text
12 34 56 78
```

甚至完全可以规定成：

```text
"305419896"
```

也可以规定成 Base64、TLV、DER。

所以：**数据的“意义”和数据的“编码表示”不是同一件事。**

这个思想其实贯穿了我们整套课程。

### 十、把前六章串起来

现在你应该已经能看到整套课程为什么这样安排了：

```text
Bit
 ↓
Byte
 ↓
Hex
 ↓
Signed / Unsigned
 ↓
Endian
 ↓
Memory Layout
 ↓
Raw Binary
 ↓
Base64
 ↓
Frame
 ↓
Payload
 ↓
Serialization
```

前面一直在研究：

> Byte 是怎么来的？
>

现在开始研究：

> **多个有意义的数据字段怎样组织成 Byte？**
>

这就是第 6 章。

这一课最核心的一张图

建议把这张图记住：

```text
程序内部
┌─────────────────────────┐
│                         │
│  struct / int / string  │
│                         │
└────────────┬────────────┘
             │
             │ Serialization
             ▼
┌─────────────────────────┐
│                         │
│   Byte Sequence         │
│                         │
│ 01 00 04 12 34 56 78   │
│                         │
└────────────┬────────────┘
             │
             │ UART / TCP / File
             ▼
┌─────────────────────────┐
│                         │
│   Byte Sequence         │
│                         │
└────────────┬────────────┘
             │
             │ Deserialization
             ▼
┌─────────────────────────┐
│                         │
│  struct / int / string  │
│                         │
└─────────────────────────┘
```

### 本课小结

第 6 章真正研究的是：

```text
有意义的数据
    ↓
如何编码
    ↓
成为 Byte
```

而 Serialization 的本质，可以浓缩成一句话：

> **把程序内部的数据表达，转换成协议约定的外部 Byte 表达。**
>

这一课我们先学了最简单的：

```text
Fixed Layout
```

但是它存在一个明显的问题：

```text
字段主要靠固定位置识别
        ↓
协议扩展比较麻烦
```

于是自然就会产生一个想法：

> 如果每个字段自己告诉接收端“我是谁、我有多长、我的值是什么”，是不是就更灵活？
>

答案就是：

```text
Tag
Length
Value
```

也就是下一课：

---

## 第 6 章 · 第 2 课时

> **TLV：Tag / Length / Value 到底解决了什么问题**
>

上一课我们已经明确了一个核心事实：

**程序里的 struct，不等于协议里的 Wire Format。**

最简单的序列化方式是 Fixed Layout：

```text
Byte 0      cmd
Byte 1~2    length
Byte 3~6    value
```

它简单、快、适合 MCU。

但问题也很明显：**字段的位置被写死了。**

这一课我们就进入一种非常重要的二进制组织方式：**TLV**

也就是：

```text
T = Tag
L = Length
V = Value
```

### 一、Fixed Layout

上节我们已经了解了

```text
Offset  Size  Field
0       1     status
1       4     counter
5       2     voltage
```

收到：

```text
01 00 00 00 64 0C E4
```

接收端知道（协议提前规定）：

```text
01          -> status
00 00 00 64 -> counter
0C E4       -> voltage
```

也就是说：**字段的“身份”，来自它所在的位置。**

而 TLV 换了一种思路：

> **字段自己携带自己的身份。**
>

最基础形式：

```text
+------+--------+----------------+
| Tag  | Length |     Value      |
+------+--------+----------------+
```

比如：

```text
01 01 05

# 我们规定：
#     Tag    = 0x01
#     Length = 0x01
#     Value  = 0x05
#     协议定义：Tag 0x01 = device_type

# 那么这三个字节表示：device_type = 5
```

Tag 的作用就是：**告诉接收方 这个字段是谁。**

例如我们定义：

```text
0x01 = Device Type
0x02 = Device ID
0x03 = Firmware Version
0x04 = Status
0x05 = Data
```

那么：

```text
04 01 01

# 可以解释成：
#     Tag    = 04
#     Length = 01
#     Value  = 01
```

根据协议：

```text
Tag 04 = Status
```

**所以：****Status = 1**

因此 Tag 本质上就是：

```text
Field ID
字段编号
字段类型标识
```

Length 是干什么的？

> 告诉解析器：**后面的 Value 占多少 Byte。**
>

Value 才是真正的业务数据

Value 可以是：

```text
一个整数
一个字符串
一串二进制
一个密钥
一个时间戳
一个设备 ID
甚至另一个 TLV 结构
```

这点非常重要。

TLV 并不规定：Value 一定是什么。

TLV 只规定外壳：

```text
Tag + Length + Value
```

至于 Value 如何解释，要看：**Tag 的定义**

### 二、TLV 和 Fixed Layout 区别

Fixed Layout：

```text
字段是谁？
    ↓
看位置（offset）
```

TLV：

```text
字段是谁？
    ↓
看 Tag
```

可以直接对比：

|Fixed Layout|TLV|
|---|---|
|靠 offset 识别字段|靠 Tag 识别字段|
|字段位置固定|字段可以更灵活|
|数据紧凑|多了 Tag/Length 开销|
|解析简单|解析稍复杂|
|扩展较麻烦|扩展更方便|

### 三、为什么 TLV 更容易扩展？

假设旧设备只认识：

```text
01 = Device Type
02 = Device ID
03 = Status
```

后来新版协议增加：**10 = Temperature**

新版设备发送：

```text
01 01 05
02 04 12 34 56 78
10 02 00 FA
03 01 01
```

老设备解析到：

```text
10 02 00 FA
```

发现：**Tag = 0x10 不认识。**

但它还有：**Length = 2**

于是它可以做：**跳过接下来的 2 个 Byte**

继续读：

```text
03 01 01
```

于是旧设备仍然能继续解析它认识的字段。

这就是 TLV 非常重要的一点：

> **即使不认识 Tag，只要 Length 是可信的，也可以跳过未知字段。**
>

这就是协议兼容性的来源之一

假设解析器逻辑：

```c
while (remain > 0)
{
    tag = *p++;
    len = *p++;

    switch (tag)
    {
        case TAG_DEVICE_TYPE:
            ...
            break;

        case TAG_DEVICE_ID:
            ...
            break;

        case TAG_STATUS:
            ...
            break;

        default:
            /* 未知字段 */
            break;
    }

    p += len;
    remain -= 2 + len;
}
```

最关键的是：

```c
default:
```

未知字段可以不理解，但因为有：**Length**

仍然知道：

```text
这个字段在哪里结束
```

于是不会把后面的字段解析乱掉。

这和第 5 章的 Length 思想其实一样

你会发现，第 5 章的 Frame：

```text
Header | Length | Payload | CRC
```

Length 解决的是：**这一帧有多长。**

TLV 里的 Length：

```text
Tag | Length | Value
```

解决的是：**这个字段有多长。**

本质思想完全一样：

```text
不知道内容具体是什么没关系
只要知道边界
就能正确移动解析位置
```

这是二进制协议解析里极其重要的思想：**边界感**

### 四、TLV 解析器本质就是移动指针

假设：

```text
01 01 05 02 04 12 34 56 78 03 01 01
^
p
```

第一次：

```text
tag = 01
len = 01

# 1 + 1 + 1 = 3
# p += 3
```

来到：

```text
02 04 12 34 56 78
^
p
```

第二项：

```text
tag = 02
len = 04

# 1 + 1 + 4 = 6
# p += 6
```

来到：

```text
03 01 01
^
p
```

所以 TLV parser 的核心其实就是：

```text
读 Tag
 ↓
读 Length
 ↓
检查 Length
 ↓
处理 Value
 ↓
移动到下一个 TLV
```

### 五、工业解析器绝对不能只写 p += len

因为 Length 是外部输入，若收到恶意或错误数据：

```text
02 FF 12 34

# Length = 255
```

但 buffer 后面只有 2 个字节，如果直接：

```c
p += len;
```

就越界了。

因此真正解析一定要

```text
# 先验证：
剩余长度 >= TLV Header

# 再验证：
剩余长度 >= Length
```

一个更正确的 TLV Parser 框架

假设：

```text
Tag    = 1 Byte
Length = 1 Byte
```

可以这样写：

```c
int tlv_parse(const uint8_t *buf, size_t len)
{
    size_t offset = 0;

    while (offset < len)
    {
        if (len - offset < 2)
        {
            return -1;
        }

        uint8_t tag = buf[offset++];
        uint8_t value_len = buf[offset++];

        if (len - offset < value_len)
        {
            return -2;
        }

        const uint8_t *value = &buf[offset];

        /* 根据 tag 处理 value */

        offset += value_len;
    }

    return 0;
}
```

最重要的两道边界检查：

```text
if (len - offset < 2)

# 以及

if (len - offset < value_len)
```

**为什么推荐 len - offset，而不是 offset + len**

你以后写安全解析器时，建议养成这种习惯：

```c
if (remaining < value_len)
```

而不是：

```c
if (offset + value_len > total_len)
```

因为整数加法存在溢出的理论风险。

更工业化一点通常维护：

```c
const uint8_t *p;
size_t remain;
```

例如：

```c
while (remain > 0)
{
    if (remain < 2)
        return ERROR;

    tag = p[0];
    len = p[1];

    p      += 2;
    remain -= 2;

    if (remain < len)
        return ERROR;

    value = p;

    p      += len;
    remain -= len;
}
```

这个模式你以后解析：

```text
TLV
DER
网络报文
文件格式
```

都会反复用到。

### 六、TLV 的 Value 可以是多字节整数

例如：

```text
02 04 12 34 56 78
```

不要直接：

```c
uint32_t value = *(uint32_t *)&buf[2];
```

原因我们已经学过：

```text
Endian
Alignment
Strict Aliasing

# 完整的字段定义通常需要：
# Tag
# Length Rule
# Value Type
# Value Encoding
```

如果协议规定 Big Endian，应显式解码：

```c
uint32_t value =
    ((uint32_t)value_ptr[0] << 24) |
    ((uint32_t)value_ptr[1] << 16) |
    ((uint32_t)value_ptr[2] << 8)  |
    ((uint32_t)value_ptr[3]);
```

所以：**TLV 只解决字段边界和字段身份。**

**TLV 还可以嵌套**

这是一个非常重要的能力，比如：

```text
Tag 0x10 = Device Info
```

它的 Value 本身不是普通数据，而是另一组 TLV：

```text
10 09
   01 01 05
   02 04 12 34 56 78
```

外层：

```text
Tag    = 10
Length = 09
Value  = ...
```

Value 内部：

```text
01 01 05
02 04 12 34 56 78
```

又是两个 TLV。

结构：

```text
Device Info
│
├─ Device Type
│
└─ Device ID
```

也就是：

```text
TLV
└─ Value
   ├─ TLV
   └─ TLV
```

这叫：**Nested TLV **嵌套 TLV。

**这已经开始非常接近 ASN.1 DER 了**

你下一课会发现，DER 本质上也大量采用：

```text
Tag
Length
Value
```

而且：**Value** 还可以包含 **另一个 TLV**

比如：

```text
SEQUENCE
│
├─ INTEGER
├─ OCTET STRING
└─ BIT STRING
```

它在 byte 层面就是：

```text
TLV
└─ Value
   ├─ TLV
   ├─ TLV
   └─ TLV
```

所以我们现在先学普通 TLV，是为了后面看 DER 时不会突然觉得陌生。

### 七、TLV 的优点

#### 1. 自描述能力更强

不是完全自描述，但至少：

```text
Tag 告诉你字段身份
Length 告诉你字段边界
```

#### 2. 扩展方便

增加：

```text
Tag 0x10
Tag 0x11
Tag 0x12
```

旧版本可以跳过。

#### 3. 可选字段自然

某个字段没有：

```text
不发送对应 TLV
```

即可。

Fixed Layout 通常还要设计：

```text
valid flag
特殊值
bitmap
```

TLV 更自然。

#### 4. 字段顺序可以更灵活

理论上：

```text
01 TLV
02 TLV
03 TLV
```

和：

```text
03 TLV
01 TLV
02 TLV
```

都可以解析。

前提是协议明确：

> 字段顺序不具有语义。
>

### 八、TLV 的典型缺点

主要有：

```text
额外 Tag/Length 开销
解析器更复杂
需要维护 Tag 定义
需要严格做边界检查
嵌套过深时实现更复杂
```

所以如果你有一个极简单协议：

```text
固定 8 Byte
永远不会升级
同一种 MCU
非常高频
```

Fixed Layout 完全可能比 TLV 更合适。

工程上不存在：**TLV 一定比 Fixed Layout 高级。**

只有：**哪个更适合你的协议。**

### 九、Tag 能不能重复

**这需要看协议定义，**例如你可以规定：

```text
Tag 01 只能出现一次
```

也可以规定：

```text
Tag 20 可以出现多次
```

比如传多个证书：

```text
20 05 .....
20 05 .....
20 05 .....
```

就相当于：`Certificate[]`

所以 TLV parser 除了解析字节，还要验证：

```text
字段是否允许重复
字段是否必须出现
字段长度是否合法
字段组合是否合法
```

这才是完整的协议解析。

#### 遇到未知 Tag 的两种策略

**（1）Strict**

```text
遇到未知 Tag
↓
报错
```

适合：

```text
安全敏感
格式必须严格一致
协议封闭
```

**（2）Extensible**

```text
遇到未知 Tag
↓
根据 Length 跳过
```

适合：

```text
协议持续升级
需要前向兼容
扩展字段
```

这个设计必须在协议层面决定，而不是 parser 随便决定。

### 十、length 的三层长度检查

我们之前理解 length 是非常危险的，永远不能直接相信它

工业上一般做三层长度检查，比如解析 SM4 Key：

```text
# 协议规定
Tag = 0x03
Length 必须 = 16
```

那么应该至少检查：

**第一层**

buffer 是否还有 TL Header：

```c
if (remain < 3)
    return ERR_TRUNCATED;
```

假设：

```text
Tag = 1 Byte
Length = 2 Byte
```

**第二层**

Length 是否超过剩余 buffer：

```c
if (value_len > remain)
    return ERR_TRUNCATED;
```

**第三层**

这个字段自身长度是否合法：

```c
if (tag == TAG_SM4_KEY && value_len != 16)
    return ERR_BAD_LENGTH;
```

这三层含义不同：

```text
Buffer 边界
协议边界
字段语义
```

不要混在一起。

### 十一、分层结构

一个非常常见的设计是：

```text
Frame
├─ Header
├─ Frame Length
├─ Command
├─ Payload
│   ├─ TLV
│   ├─ TLV
│   └─ TLV
└─ CRC

# 比如：
# AA 55
# 00 0C
# 10
# 01 01 05
# 02 04 12 34 56 78
# CRC
```

Frame Parser 先负责：

```text
从 byte stream 中找出完整 Frame
```

然后 Command Handler 负责：

```text
识别命令
```

最后 TLV Parser 负责：

```text
解析 Payload 内部字段
```

三个层次：

```text
Byte Stream
    ↓
Frame Layer
    ↓
Command Layer
    ↓
TLV / Serialization Layer
```

这是非常典型的协议分层。

**代码结构也分层**

不要把所有逻辑写在：

```c
uart_rx_callback()
```

更合理的是：

```text
UART RX
   ↓
Frame Parser
   ↓
Command Dispatcher
   ↓
TLV Parser
   ↓
Business Logic
```

例如：

```c
frame_parse(...)
```

得到：

```c
frame.command
frame.payload
frame.payload_len
```

然后：

```c
handle_set_config(
    frame.payload,
    frame.payload_len);
```

里面：

```c
tlv_parse(...)
```

再转成：**Config config;**

最后业务层只操作：

```c
config.device_id
config.status
config.key
```

而不用关心原始 byte。

这就是协议软件设计中非常重要的：**分层。**

### 十二、这一课与下一课的连接

现在我们已经知道：

```text
01 01 05
02 04 12 34 56 78
```

是一种 TLV。

但是问题来了：**Tag 到底怎样定义？**

比如：

```text
01 是 INTEGER？
02 是 STRING？
03 是 SEQUENCE？
```

不同协议可以自己随便规定。

于是就有人做了一件事情：

> 不再让每个项目自己随便定义数据类型，
>
> 而是建立一套正式的数据结构描述语言。
>

这就是：**ASN.1**

ASN.1 负责描述：

```text
这个东西是 INTEGER
这个东西是 OCTET STRING
这个东西是 SEQUENCE
这个东西里面还有哪些字段
```

然后：

```text
BER / DER
```

再规定：**这些 ASN.1 类型到底怎样变成 Byte。**

于是：

```text
ASN.1
   ↓
BER / DER
   ↓
TLV Byte Sequence
```

---

## 第 6 章 · 第 3 课时

前两课我们已经完成了这条主线：

```text
程序内部数据
    ↓
Serialization
    ↓
Fixed Layout / TLV
```

认识了 TLV：**Tag \| Length \| Value**

现在进入一个非常重要、也非常容易混乱的领域：

```text
ASN.1
BER
DER
```

很多人第一次看到这些名字，会下意识认为：

> ASN.1、BER、DER 都是一种“编码格式”。
>

其实不是。

这一课最重要的任务，就是把这三个概念彻底分开。

### 一、先给结论

可以先记住：

```text
ASN.1
=
描述“数据结构是什么”
```

而：

```text
BER / DER
=
规定“这些数据结构最终怎样编码成 Byte”
```

也就是：

```text
ASN.1
      数据结构描述
           │
           ▼
    Encoding Rules
       编码规则
           │
     ┌─────┴─────┐
     ▼           ▼
    BER         DER
     │           │
     └─────┬─────┘
           ▼
      Byte Sequence
```

所以：**ASN.1 不是最终那串 Byte。**

它更像是一门：**数据结构描述语言。**

**为什么需要 ASN.1？**

假设我们自己设计一个协议，例如设备身份信息：

```text
DeviceInfo
│
├─ version
├─ serialNumber
└─ publicKey
```

当然可以在文档里写：

```text
version:
    uint8_t

serialNumber:
    string

publicKey:
    binary
```

但马上会出现大量问题：

```text
version 到底多大？
signed 还是 unsigned？

string 是 ASCII 还是 UTF-8？

publicKey 是固定 64 Byte？
还是可变长度？

字段有没有顺序？

字段能不能省略？

能不能有数组？

能不能嵌套？
```

如果每个协议都靠人类自然语言描述，很容易产生歧义。

于是我们需要一种：

> **正式、严格、机器也能够理解的数据结构描述语言。**
>

ASN.1 就是干这个的。

### 二、ASN.1 全称

ASN.1：**Abstract Syntax Notation One（抽象语法标记一）**

名字其实已经透露了它的性质：

```text
Abstract Syntax
```

也就是：

> 抽象描述“数据长什么样”。
>

而不是直接规定：

```text
30 0D 02 01 ...
```

这些具体 Byte。

**看一个最简单的 ASN.1**

比如我们想定义一个人：

```text
Person
│
├─ name
└─ age
```

ASN.1 可以写成：

```text
Person ::= SEQUENCE {
    name UTF8String,
    age  INTEGER
}
```

先不要管语法细节。

你只要看懂：

```text
Person
```

是一个结构，它里面：

```text
name    是：UTF8String

age     是：INTEGER
```

**注意：这里仍然没有 Byte**

ASN.1：

```text
Person ::= SEQUENCE {
    name UTF8String,
    age  INTEGER
}
```

只描述：

```text
Person 是一个 SEQUENCE
里面有 name
里面有 age
```

它并没有告诉你：

```text
name 前面是不是 0x0C？
age 是不是 4 Byte？
Length 占几个 Byte？
整数是不是 Big Endian？
```

这些都不是 ASN.1 类型定义本身负责解决的。

真正把它变成 Byte，需要：**Encoding Rules，也就是编码规则。**

**这跟 C struct 有一点像，但又不一样**

比如 C：

```c
typedef struct
{
    char name[32];
    int age;
} Person;
```

它也是在描述：

```text
Person 有哪些成员
```

但 C struct 是为了：**程序运行时内存。**

ASN.1 是为了：**交换数据的抽象结构描述。**

可以粗略对比：

|C|ASN.1|
|---|---|
|`struct`|`SEQUENCE`|
|`int`|`INTEGER`|
|byte array|`OCTET STRING`|
|bool|`BOOLEAN`|

不要把它们完全等同。

### 三、ASN.1 解决的是“语义层”

我们可以把问题分三层。

**第一层：数据的意义**

例如：

```text
age = 18
```

**第二层：数据结构**

例如：

```text
Person
├─ name
└─ age
```

ASN.1 主要工作在这里。

**第三层：Byte 表达**

例如最后编码成：

```text
30 0A ...
```

BER / DER 负责这里。

所以：

```text
Meaning
   ↓
ASN.1 Structure
   ↓
Encoding Rules
   ↓
Bytes
```

### 四、ASN.1 理解点

#### （1）ASN.1 有很多基本类型

后面 DER 实战时我们还会详细看，现在先建立印象。

常见 ASN.1 类型包括：

```text
BOOLEAN
INTEGER
BIT STRING
OCTET STRING
NULL
OBJECT IDENTIFIER
UTF8String
PrintableString
SEQUENCE
SET
```

密码学里尤其常见：

```text
INTEGER
BIT STRING
OCTET STRING
OBJECT IDENTIFIER
SEQUENCE
```

以后你看：

```text
公钥
私钥
证书
签名
算法标识
```

会大量遇到。

#### （2）SEQUENCE 特别重要

例如：

```text
Person ::= SEQUENCE {
    name UTF8String,
    age  INTEGER
}
```

意思就是：

```text
Person
│
├─ name
└─ age
```

也就是一个：**复合结构。**

你可以把它类比成：**`struct`**** **虽然不完全相同。

密码学里很多结构都是：

```text
SEQUENCE
```

里面嵌套：

```text
INTEGER
OID
BIT STRING
SEQUENCE
...
```

#### （3）ASN.1 还可以定义可选字段

比如：

```text
Person ::= SEQUENCE {
    name    UTF8String,
    age     INTEGER,
    address UTF8String OPTIONAL
}
```

这里 **`OPTIONAL`****，**表示：

> address 可以存在，也可以不存在。
>

这说明 ASN.1 不只是定义数据类型，还可以定义：

```text
字段约束
可选性
嵌套关系
集合关系
```

因此它非常适合复杂协议。

#### （4）ASN.1 也可以定义枚举

例如：

```text
Status ::= ENUMERATED {
    idle    (0),
    running (1),
    error   (2)
}
```

于是逻辑意义：

```text
0 = idle
1 = running
2 = error
```

比仅仅写：**`uint8_t status;`**** **表达力更强。

#### （5）ASN.1 是“Schema”

这里正式引入一个很重要的词：**Schema**

> 可以理解为：**数据结构说明书。**
>

例如：

```text
Person ::= SEQUENCE {
    name UTF8String,
    age INTEGER
}
```

就是：**Person 的 Schema**

类似：

```text
数据库表结构
JSON Schema
Protobuf .proto
```

都是在描述：*数据应该长什么样。*

### 五、BER 是什么？

现在 ASN.1 只告诉我们：

```text
这是 INTEGER
这是 STRING
这是 SEQUENCE
```

但是最终要传输，总得变成 Byte，于是出现：BER

全称：**Basic Encoding Rules** 基本编码规则。

BER 规定：

> ASN.1 中定义的数据，应该怎样编码成具体 Byte。
>

**BER 大量使用 TLV 思想**

这就是为什么我们上一课专门先讲 TLV。

BER 编码的基本思想就是：

```text
Identifier
Length
Contents
```

通常我们也可以理解成：

```text
Tag
Length
Value

# 即
# T
# L
# V
```

所以当你以后看到 BER/DER：

```text
30 03 02 01 05
```

它本质仍然是在做：

```text
Tag
Length
Value
```

只是规则比我们自己设计的 TLV 更严格、更复杂。

**举一个简单概念例子**

假设 ASN.1 逻辑值：

```text
INTEGER 5
```

BER/DER 编码可能类似：

```text
02 01 05
```

拆开：

```text
02 | 01 | 05
 T    L    V
```

其中

- `02` 表示 **INTEGER**
- `01` 表示 **Value 长度 = 1 Byte**
- `05` 就是 **整数 5**

是不是突然很熟悉？因为：

> BER / DER 本质依然是 TLV。
>

### 六、SEQUENCE 也可以编码成 TLV

比如逻辑结构：

```text
SEQUENCE {
    INTEGER 5
}
```

内部 INTEGER：**02 01 05**

外面再包一个 **SEQUENCE：**

```text
30 03
   02 01 05
```

也就是：

```text
30 | 03 | 02 01 05
 T    L       V
```

这里外层 Value：

```text
02 01 05
```

本身又是：**一个完整 TLV。**

所以你就能看见：

```text
TLV
└─ Value
   └─ TLV
```

这就是我们上一课讲的：**Nested TLV**

### 七、BER 会出现多个合法编码

这就是 BER 和 DER 最核心的区别之一。

BER 的目标之一是：**灵活。**

因此同一个 ASN.1 值，在某些情况下可以有：**多种合法编码方式。**

也就是说：

```text
同一个逻辑数据
```

可能对应：

```text
Byte Sequence A

# 或者
Byte Sequence B
```

两者解析后：逻辑意义完全一样

**这对普通通信可能没问题**

假设：

```text
Encoding A  和  Encoding B
```

最后都能还原：**age = 18**

对于普通通信来说：只要能正确解析，就可能没问题。

**但是到了密码学领域，就有大问题。**

**数字签名最怕“同义不同 Byte”**

假设逻辑数据：

```text
Data
```

可以编码成：

```text
Bytes A  或者  Bytes B
```

而：

```text
Decode(Bytes A)
=
Decode(Bytes B)
```

逻辑意义一样。

但密码学签名通常签的是：**Byte Sequence**

比如：`Hash(Bytes A)` 和 `Hash(Bytes B)` 几乎一定不同。

那么：

```text
Signature(Bytes A)
```

就不能直接验证：**Bytes B**

所以密码学非常希望：*同一个逻辑对象只能有一种标准 Byte 表达。*

**这就是 DER 的意义。**

### 八、DER

DER：**Distinguished Encoding Rules（**唯一编码规则 / 可区别编码规则**）**

更容易理解的说法是：**BER 的严格、规范化子集。**

关系：

```text
BER
│
├─ 允许多种合法表示
│
└─ DER
    └─ 强制选择唯一、规范的一种表示
```

所以：

```text
DER ⊂ BER

# 所有合法 DER 都符合 BER。
# 合法 BER 不一定符合 DER。
```

**DER 的核心思想：Canonical Encoding**

这是以后非常常见的词，意思是：**规范化编码 / 唯一编码。**

核心要求：

```text
同一个 ASN.1 值
      ↓
只能得到
      ↓
唯一的 DER Byte Sequence
```

可以理解为：

```text
ASN.1 Object
      ↓
DER
      ↓
唯一 Byte 表达
```

**为什么数字证书特别喜欢 DER？**

比如 X.509 证书，证书内部包含：

```text
Version
Serial Number
Issuer
Subject
Public Key
Validity
Extensions
Signature Algorithm
Signature
```

这是非常复杂的数据结构，ASN.1 很适合描述它。

但是签名需要稳定的 Byte。于是：

```text
ASN.1
+
DER
```

就成为非常自然的组合。

**以后你会大量见到：**

```text
ASN.1 structure
        ↓
DER encoded data
```

比如：

```text
X.509 Certificate
PKCS#8 PrivateKeyInfo
SubjectPublicKeyInfo
ECDSA Signature
SM2 Signature
RSA Public Key
```

都可能涉及 ASN.1 / DER，这也是为什么我们专门安排这一章。

### 九、ASN.1 和 DER 不要混成一个词

工程中经常有人说：**“这是 ASN.1 格式。”**

这句话很多时候只是口语化说法，更严谨应该问：

```text
ASN.1 描述的是什么结构？
```

以及：

```text
使用什么 Encoding Rules？
BER？
DER？
PER？
```

因为：**ASN.1 本身不是唯一的一种 Byte 编码**。

**ASN.1 其实还有其他 Encoding Rules**

除了：

```text
BER
DER
```

还有其他编码规则，例如：

```text
CER
PER
XER
等等
```

我们这门课不会展开，因为你的嵌入式/密码学场景最重要的是：

```text
ASN.1
BER
DER
```

尤其：**DER**

**所以完整关系应该这样看**

错误理解：

```text
ASN.1
BER
DER
=
三个平级格式
```

正确理解：

```text
ASN.1
数据结构描述语言
        │
        ▼
Encoding Rules
        │
        ├─ BER
        ├─ DER
        ├─ CER
        └─ ...
```

其中：

```text
DER 是 BER 的严格子集
```

### 十、举类比

可以类比成：

```text
ASN.1
≈
建筑设计图
```

它告诉你：

```text
这里是客厅
这里是卧室
这里是厨房
```

但是没有规定：

```text
砖具体怎么编号运输
```

而：

```text
BER / DER
# 如何把这套设计图变成一套明确的施工编码规则。
```

其中 *BER*：**允许一些不同但合法的施工方式**

*DER*：**强制采用唯一标准施工方式**

**再类比 Protobuf**

你以后接触 Protobuf 时会看到：

```text
message Person {
    string name = 1;
    int32 age = 2;
}
```

这个 `.proto` 很像：**Schema。**

ASN.1：

```text
Person ::= SEQUENCE {
    name UTF8String,
    age INTEGER
}
```

也属于这种思想：

> 先描述数据结构。然后再通过特定编码规则变成 Byte。
>

所以：

```text
ASN.1
```

并不是一个很神秘的密码学专属东西。

它的本质还是：**结构化数据描述 + 编码。**

### 十一、密码学里一个非常典型的 ASN.1

比如 ECDSA / SM2 风格的签名，经常逻辑上包含：

```text
r
s
```

可以用 ASN.1 描述成：

```text
Signature ::= SEQUENCE {
    r INTEGER,
    s INTEGER
}
```

逻辑结构：

```text
Signature
│
├─ r
└─ s
```

DER 编码后则变成：

```text
SEQUENCE TLV
│
└─ Value
   ├─ INTEGER TLV
   └─ INTEGER TLV
```

即：

```text
30 LL
   02 LL rr...
   02 LL ss...
```

你现在不需要马上会算具体 Length，下一课我们专门做。

但现在应该已经能理解结构了。

**这也解释了你以后看到的 ****`30`**

如果你以前见过类似：

```text
30 44
02 20 ...
02 20 ...
```

可能会觉得：

```text
30 是什么？
44 又是什么？
02 又是什么？
```

现在至少可以先建立框架：

```text
30
≈ SEQUENCE 的 Tag

44
≈ 后面 Value 的长度

02
≈ INTEGER 的 Tag
```

于是：

```text
30 44
   02 ...
   02 ...
```

就是：

```text
SEQUENCE
├─ INTEGER
└─ INTEGER
```

是不是比以前清晰很多。

### 十二、PEM 又在哪里？

你一开始学习这套课程，就是因为碰到了：

```text
PEM
ASN.1
Base64
DER
```

现在可以先把它们放到正确位置：

```text
ASN.1
   ↓
定义数据结构

DER
   ↓
编码成二进制 Byte

Base64
   ↓
把二进制转换成可打印文本

PEM
   ↓
再加 BEGIN/END 头尾
```

例如：

```text
ASN.1 Object
     ↓
DER
     ↓
Binary
     ↓
Base64
     ↓
-----BEGIN ...
...
-----END ...
```

所以：

```text
PEM 不是 ASN.1。

Base64 也不是 ASN.1。

DER 也不等于 PEM。
```

它们处在不同层。

### 十三、比如一个证书

逻辑流程：

```text
X.509 Certificate
       │
       ▼
ASN.1 定义结构
       │
       ▼
DER 编码
       │
       ▼
二进制证书
       │
       ▼
Base64
       │
       ▼
加 PEM Header/Footer
```

于是得到：

```text
-----BEGIN CERTIFICATE-----
MIIB...
...
-----END CERTIFICATE-----
```

这条链你现在应该开始能看懂了。

### 十四、为什么 DER 看起来像 TLV？

因为它本质就是：

```text
Identifier
Length
Contents
```

也就是：

```text
Tag
Length
Value
```

所以我们课程安排是：

```text
Serialization
      ↓
TLV
      ↓
ASN.1
      ↓
BER / DER
```

而不是一上来就丢给你：

```text
30 82 01 22 ...
```

否则很容易纯背规则。

现在你知道：**DER 不是突然冒出来的一套神秘 Hex。**

它仍然建立在：

```text
字段身份
字段长度
字段内容
```

这三个基本思想之上。

**BER 和 DER 的核心差别**

BER

```text
只要编码合法、能表达 ASN.1 数据即可
```

DER

```text
不仅合法，而且必须使用唯一规范形式
```

所以 DER 特别适合：

```text
数字签名
证书
密钥
需要稳定哈希结果的数据
```

### 十五、一个很重要的理解

DER 的“唯一”不是说：

> 世界上所有不同数据最后编码都一样。
>

而是：**同一个逻辑 ASN.1 值只能有一个合法 DER 编码。**

也就是：

```text
Object A
   ↓
唯一 DER A
```

另一个对象：

```text
Object B
   ↓
唯一 DER B
```

这是 Canonical 的含义。

**这一课暂时不要纠结 Tag Byte 细节**

比如以后我们会看到：

```text
02
03
04
05
06
30
```

分别可能对应：

```text
INTEGER
BIT STRING
OCTET STRING
NULL
OBJECT IDENTIFIER
SEQUENCE
```

但今天不要把注意力放在背数字上。

这一课真正要解决的是：

```text
ASN.1
BER
DER
```

三者的层级关系。

### 十六、把整个知识树串起来

现在我们的第 6 章已经走到这里：

```text
程序内部对象
      │
      ▼
Serialization
      │
      ├─ Fixed Layout
      │
      └─ TLV
           │
           ▼
        ASN.1
   描述数据结构
           │
           ▼
      BER / DER
     规定 Byte 编码
           │
           ▼
        Binary
```

再往外：

```text
DER Binary
    │
    ▼
Base64
    │
    ▼
PEM
```

这已经把你最开始困惑的很多概念串起来了。

**本课最重要的一张图**

```text
ASN.1
      “数据是什么结构”
              │
              ▼
      Encoding Rules
              │
      ┌───────┴───────┐
      ▼               ▼
     BER             DER
  灵活编码       规范唯一编码
      │               │
      └───────┬───────┘
              ▼
        TLV Byte Stream
              │
              ▼
        30 0D 02 ...
```

这种真实 Hex，一 Byte 一 Byte 地拆。

---

## 第 6 章 · 第 4 课时

DER 编码规则：Tag / Length / Value 到底怎么读

这一课开始，我们正式进入 **DER 的字节级解析**。

前一课我们已经知道：

```text
ASN.1
负责描述“数据结构是什么”

DER
负责规定“最终 Byte 怎么编码”
```

而 DER 最核心的结构仍然是：

```text
Tag | Length | Value
```

所以这一课的目标非常明确：

> **看到一串 DER Hex，能够先把每个 TLV 的边界拆出来。**
>

先不追求一次看懂复杂证书，先把基本规则吃透。

### 一、最简单的 DER：INTEGER 5

假设 ASN.1 逻辑值是：

```text
INTEGER 5
```

DER 可以编码为：

```text
02 01 05
```

拆开：

```text
02 | 01 | 05
 T    L    V

 # Tag 0x02 = INTEGER
 # 所以：02 01 05
 # 就是：INTEGER，长度 1 Byte，值为 5。
```

**DER 的基本解析步骤**

以后看到任何 DER，都先按这个顺序：

```text
1. 读 Tag
2. 读 Length
3. 根据 Length 截取 Value
4. 判断 Value 是否还包含内部 TLV
```

也就是：

```text
Tag
 ↓
Length
 ↓
Value Boundary
 ↓
Interpret Value
```

最重要的是先找边界。

不要一上来就试图理解所有字段的业务意义。

### 二、解析常见 Tag

这里最值得先记住的是：

```text
02 = INTEGER
03 = BIT STRING
04 = OCTET STRING
06 = OID
30 = SEQUENCE
```

密码学里特别常见。

**这里需要开始看 Tag Byte 的内部结构。**

一个基础 Tag Byte 可以粗略拆成：

```text
bit 7 6   bit 5   bit 4 3 2 1 0
+-------+-------+-------------+
| Class | P/C   | Tag Number  |
+-------+-------+-------------+
```

也就是：

```text
Class
Primitive / Constructed
Tag Number
```

**Class 是什么？**

最高两位：

```text
00 = Universal
01 = Application
10 = Context-specific
11 = Private
```

我们现在最常见的是：

```text
00 = Universal
```

也就是 ASN.1 内建类型：

```text
INTEGER
BIT STRING
OCTET STRING
SEQUENCE
```

这些。

**Primitive 和 Constructed**

bit 5：

```text
0 = Primitive
1 = Constructed
```

这个区别非常重要。

**Primitive**

表示 Value 本身就是数据，例如：

```text
02 01 05
```

INTEGER 的：

```text
Value = 05
```

就是数据本身。

**Constructed**

表示 Value 里面还装着别的 TLV，例如：

```text
30 03 02 01 05
```

这里 **30 **是 SEQUENCE，它的 Value：

```text
02 01 05
```

本身又是一个 INTEGER TLV。

所以：

```text
SEQUENCE
└─ INTEGER
```

这就是 Constructed。

#### （1）为什么 `0x02` 是 INTEGER？

把 **02 写成二进制：**

```text
0000 0010

# 拆：00 | 0 | 00010
```

解释：

```text
Class = 00
       = Universal

P/C   = 0
       = Primitive

Tag Number = 2
```

Universal 类型编号 2：

```text
INTEGER
```

所以：

```text
02 = INTEGER
```

#### （2）为什么 `0x04` 是 OCTET STRING？

```text
04
=
0000 0100

# 拆：00 | 0 | 00100
```

所以：

```text
Universal
Primitive
Tag Number = 4
```

于是：

```text
04 = OCTET STRING
```

#### （3）为什么 `0x30` 是 SEQUENCE？

```text
30
=
0011 0000

# 拆：00 | 1 | 10000
```

也就是：

```text
Class = Universal
P/C   = Constructed
Tag Number = 16
```

Universal Tag Number 16：

```text
SEQUENCE
```

所以：

```text
30 = SEQUENCE
```

这就是 `30` 的来源。

### 三、看一个例子

```text
30 08
   02 01 05
   04 03 41 42 43
```

先看外层：

```text
30
=
SEQUENCE

08
=
Value 长度 8 Byte
```

后面 8 Byte：

```text
02 01 05
04 03 41 42 43

# 第一个内部 TLV：02 01 05
# 表示：
#   INTEGER 5
#   占 3 Byte

# 第二个内部 TLV：04 03 41 42 43
# 表示：
#   OCTET STRING 41 42 43
#   占 5 Byte
```

如果把：41 42 43 按 ASCII 看，所以结构：

```text
SEQUENCE
├─ INTEGER 5
└─ OCTET STRING
   └─ 41 42 43
```

Length 不一定只有 1 Byte，如果 Length 小于 128：

直接用一个 Byte 表示，这叫 **Short Form，**例如：

```text
Length：0 ~ 127

00 ~ 7F
```

为什么不能直接用 `80` 表示 128？

因为 Length Byte 的最高位有特殊含义，规则：

```text
最高位bit = 0
→ Short Form
```

即假设用 `80` ：

```text
1000 0001
```

最高位为 1，就进入：**Long Form**

### 四、Long Form Length

如果第一个 Length Byte：

```text
81

# 1 | 0000001
```

意思不是 Length = 129，而是：

> 后面还有 1 Byte，用来表示真正 Length。
>

比如：

```text
81 80
```

表示：

```text
后面 1 Byte 表示 Length
Length = 0x80
       = 128
```

再比如 `82`表示：**后面 2 Byte 是 Length。**

```text
82 01 00
```

那么：

```text
Length = 0x0100
       = 256
```

### 五、DER 对 Length 要求

DER 是 Canonical Encoding，所以：

> 能用短形式，就不能故意用长形式。
>

比如 Length = 5。

```text
05
```

不应该编码成：

```text
81 05
```

因为虽然某些更宽松编码规则可能能表达相同含义，但 DER 要求：

> 使用唯一、最短的规范形式。
>

这就是 DER 比 BER 严格的一个例子。

**DER 中不会使用 BER 的 indefinite length**

BER 某些情况下允许“不定长”形式，例如：

```text
80

# indefine length, 意思是这个长度不预先给出
# 解码器必须一直读到 End-of-Content(EOC) 标记（0x00 0x00）才结束
```

但 DER：**不允许 indefinite length。**

DER 必须明确知道：

```text
Value 到底有多长
```

所以 DER 总是使用：**definite length。**

这也是 DER 更适合规范化编码的原因之一。

### 六、OCTET STRING

例如：

```text
04 04 DE AD BE EF

# 04          = OCTET STRING
# 04          = Length = 4
# DE AD BE EF = 4 Byte Value
```

所以：

```text
OCTET STRING = DE AD BE EF
```

这里 Value 不一定是文本，它就是：**一串原始 Byte。**

这对密码学非常重要。

比如：

```text
密钥
随机数
哈希
IV
```

经常会被装进：

```text
OCTET STRING
```

OCTET STRING 和普通字符串不要混淆

`OCTET STRING`：

```text
04 ...
```

本质是：**Byte Array。**

不代表：

```text
ASCII
UTF-8
```

比如：

```text
04 03 FF 00 A5
```

完全合法地表达：

```text
FF 00 A5
```

这三个 Byte。

初步可以这样理解：

```text
OCTET STRING
≈ byte[]
```

### 七、BIT STRING

`BIT STRING`：

```text
03 03 00 AA BB

# 03 = BIT STRING
# 03 = Length = 3
# value：00 AA BB
```

这里有一个非常特殊的规则：

> BIT STRING 的 Value 第一个 Byte，不是实际业务数据，而是 unused bits 数量。
>

什么叫 unused bits？

BIT STRING 允许数据不是 8 的整数倍 bit，比如只有：

```text
13 bits
```

最后一个 Byte 不可能全部有效。

于是要记录：

```text
最后一个 Byte 有几个 bit 没用
```

这个数量放在 Value 的第一个 Byte。

最常见情况：`00`

例如：

```text
00 AA BB
```

其中：

```text
00
=
unused bits = 0
```

真正 bit 数据：`AA BB`

也就是说：

```text
16 bit 全部有效
```

所以解析 BIT STRING 时千万不要直接认为：

```text
Value = AA BB
```

应该先读：**unused bits**

这在公钥里特别常见

比如 SubjectPublicKeyInfo 里的公钥，经常是：

```text
BIT STRING
```

可能看到类似：

```text
03 42 00 04 XX XX XX ...
```

含义：

```text
03
=
BIT STRING

42
=
Length = 0x42

00
=
unused bits = 0

04 ...
=
真正的公钥数据
```

这里常常让初学者困惑：

> 公钥为什么前面多了一个 00？
>

现在就知道：**那可能不是公钥本身，而是 BIT STRING 的 unused-bits 字段。**

### 八、NULL

NULL 的典型 DER：

```text
05 00
```

就是 **NULL** ，因为：

```text
Tag    = 05
Length = 00
Value  = 无
```

非常简单。

### 九、OBJECT IDENTIFIER

OID：

```text
Tag = 06
```

比如：

```text
06 03 2A 03 04
```

表示：

```text
OBJECT IDENTIFIER
```

具体 OID 的 Value 编码规则稍复杂。

这一章我们不展开到完整算法。

先记住：

```text
06
=
OID
```

而 OID 在密码协议里非常常见，因为它用来标识：

```text
算法
曲线
哈希算法
签名算法
证书属性
```

例如某个结构里：

```text
AlgorithmIdentifier
```

通常就会包含：**OBJECT IDENTIFIER**

### 十、INTEGER 有一个非常关键的坑

你可能以为：

```text
02 01 80
```

表示整数：128

其实不是，ASN.1 INTEGER 是：**有符号整数。**

并采用二进制补码语义，所以最高 bit：

```text
bit7
```

会影响正负。

`0x80` 会被看成负数

```text
80
=
1000 0000
```

最高位是 1，作为有符号补码：

```text
0x80
=
-128
```

所以：

```text
02 01 80
```

表示：**INTEGER -128**

**那正数 128 怎么编码？**

为了避免最高 bit 被误认为符号位，需要在前面加：

```text
00
```

所以 +128 编码为：

```text
02 02 00 80

# 02    = INTEGER
# 02    = Length = 2
# 00 80 = +128
```

这个规则对：

```text
ECDSA
SM2
```

签名中的：

```text
r
s
```

极其重要。

**为什么 SM2/ECDSA DER 签名长度会变化？**

假设：

```text
r
```

固定逻辑宽度是 32 Byte。

但如果 r 的第一个 **Byte \>=  0x80 **例如：

```text
A1 ...
```

那么 ASN.1 INTEGER 为了保证这是正数，要编码成：

```text
00 A1 ...
```

于是：*32 Byte *变成* 33 Byte*

同样：

```text
s
```

也可能需要前导 `00`，所以 DER 签名总长度不是永远固定。

这是一个非常实用的知识点。

**比如一个简化签名**

```text
r = 01 23
s = 7F 45
```

都没有最高位为 1，那么可以近似编码：

```text
30 08
   02 02 01 23
   02 02 7F 45
```

结构：

```text
SEQUENCE
├─ INTEGER r
└─ INTEGER s
```

**如果 r 开头是 ****`80`****：**

```text
r = 80 23
```

这是一个正数，如果直接：

```text
02 02 80 23
```

会被解释为负数。

所以必须：

```text
02 03 00 80 23
```

**这时 Length 从 02 变成 03**

**这就是 DER 签名长度变化的来源**

对于 256-bit 椭圆曲线：

```text
r = 32 Byte
s = 32 Byte
```

逻辑 raw signature 常常是：

```text
r || s
```

固定：**64 Byte**

但 DER：

```text
SEQUENCE {
    INTEGER r,
    INTEGER s
}
```

INTEGER 可能：**32 Byte** 也可能是 **33 Byte**

所以最终 DER 签名常见长度会浮动，这也是为什么：

> Raw SM2/ECDSA signature 和 DER signature 不能直接混为一谈。
>

### 十一、来看一个完整例子

现在解析：

```text
30 0D
   02 01 01
   04 03 41 42 43
   03 03 00 AA BB
```

先看外层：

```text
30 = SEQUENCE

0D = Length = 13 Byte
```

所以 Value 是后面的 13 Byte。

**第一个内部字段**

```text
02 01 01
```

解析：

```text
02 = INTEGER
01 = Length 1
01 = Value 1
```

所以：

```text
INTEGER = 1
```

**第二个内部字段**

```text
04 03 41 42 43
```

解析：

```text
04 = OCTET STRING
03 = Length 3
41 42 43 = Value
```

如果按 ASCII：

```text
"ABC"
```

**第三个内部字段**

```text
03 03 00 AA BB
```

解析：

```text
03 = BIT STRING
03 = Value 长度 3
00 AA BB = Value
```

其中：

```text
00 = unused bits = 0
```

真正 bit 数据：

```text
AA BB
```

**最后还原 ASN.1 结构**

最终：

```text
SEQUENCE
├─ INTEGER
│  └─ 1
│
├─ OCTET STRING
│  └─ 41 42 43
│
└─ BIT STRING
   ├─ unused bits = 0
   └─ AA BB
```

这就是一次完整 DER 手工解析。

### 十二、解析 DER 时一定养成“长度闭环”

以后解析一个 SEQUENCE，不要只是：**“看起来差不多。”**

而是一定算：

```text
所有内部 TLV 总长度
=
外层 Value Length
```

例如：

```text
30 0D
```

你就应该验证：

```text
里面是否真的正好 13 Byte
```

这是判断 DER 是否解析正确的重要方法。

### 十三、DER Parser 的本质

伪代码可以写成：

```c
parse_tlv(buf, len)
{
    parse_tag();

    parse_length();

    check(value_length);

    if (constructed)
    {
        parse_children(value, value_length);
    }
    else
    {
        parse_primitive_value();
    }
}
```

所以：**DER Parser 本质就是一个递归 TLV Parser。**

**和上一课 TLV 完全接上了**

普通 TLV：

```text
Tag
Length
Value
```

DER：

```text
Tag
Length
Value
```

区别只是 DER 对：

```text
Tag 编码
Length 编码
Value 编码
类型语义
唯一性
```

都做了严格规范。

所以你可以把 DER 理解成：

> **高度标准化、类型化、规范化的 TLV 系统。**
>

这个理解非常准确。

---

## 第 6 章 · 第 5 课时

实战：手工拆解真实 DER Hex，并串起 ASN.1 / DER / Base64 / PEM

这是第 6 章最后一课。前四课我们已经建立了完整基础：

```text
第 1 课
Serialization
程序数据 → Byte

第 2 课
TLV
Tag + Length + Value

第 3 课
ASN.1 / BER / DER
结构描述与编码规则

第 4 课
DER Tag / Length / Value
真正开始读 Byte
```

这一课不再继续堆新概念。

我们的目标只有一个：

> **拿一段接近真实密码学场景的 DER 数据，从第一个 Byte 一直解析到底。**
>

而且最终要把这几个概念彻底串起来：

```text
ASN.1
DER
Raw Binary
Hex
Base64
PEM
```

### 一、一个“签名”

假设某个 SM2 / ECDSA 风格的签名，逻辑上由两个大整数组成：

```text
Signature
├─ r
└─ s
```

ASN.1 可以描述成：

```text
Signature ::= SEQUENCE {
    r INTEGER,
    s INTEGER
}
```

注意 这只是：**ASN.1 结构描述。**

现在假设：

```text
r =
A1 01 02 03 04 05 06 07
08 09 0A 0B 0C 0D 0E 0F
10 11 12 13 14 15 16 17
18 19 1A 1B 1C 1D 1E 1F
```

```text
s =
2B 01 02 03 04 05 06 07
08 09 0A 0B 0C 0D 0E 0F
10 11 12 13 14 15 16 17
18 19 1A 1B 1C 1D 1E 1F
```

如果采用最简单的 Raw 格式：**r \|\| s**

那就是：*32 + 32 = 64 Byte*

但是 ASN.1 DER 不是直接：

```text
r || s
```

而是：

```text
SEQUENCE
├─ INTEGER r
└─ INTEGER s
```

所以需要逐层编码。

**先处理 r**

r 的第一个 Byte 是 **A1** ，二进制：

```text
1010 0001
^
最高 bit = 1
```

ASN.1 INTEGER 是有符号整数，按照我们之前的学习

最高位为 1，可能被解释为负数，所以 DER 必须加：**00**

变成：

```text
00 A1 01 02 03 ...
```

所以 r 的 INTEGER 编码：

```text
02 21 00
A1 01 02 03 04 05 06 07
08 09 0A 0B 0C 0D 0E 0F
10 11 12 13 14 15 16 17
18 19 1A 1B 1C 1D 1E 1F
```

即：

```text
02
│
└─ INTEGER

21
│
└─ Length = 33

00 A1 01 ... 1F
│
└─ Value
```

注意：前面的 `00` 不属于原始 r。

它只是：

> DER INTEGER 为了保护正数符号而添加的编码字节。
>

这一点以后做签名格式转换时非常关键。

**再处理 s**

s 第一个 Byte 是 **2B** ，二进制：

```text
0010 1011
^
最高 bit = 0
```

所以它本身已经可以明确表示正数，不需要添加：**00**

于是 s 的 INTEGER 编码：

```text
02 20
2B 01 02 03 04 05 06 07
08 09 0A 0B 0C 0D 0E 0F
10 11 12 13 14 15 16 17
18 19 1A 1B 1C 1D 1E 1F
```

即：

```text
02
=
INTEGER

20
=
Length = 32

2B 01 ... 1F
=
s
```

**把两个 INTEGER 放进 SEQUENCE**

第一个 INTEGER 总共占：

```text
Tag       1
Length    1
Value    33
--------------
总计     35 Byte
```

第二个 INTEGER：

```text
Tag       1
Length    1
Value    32
--------------
总计     34 Byte
```

因此 SEQUENCE 的 Value 总长度：

```text
35 + 34 = 69 Byte
```

69 的十六进制：**0x45**

所以外层：

```text
30 45
```

最终完整 DER：

```text
30 45
   02 21 00
      A1 01 02 03 04 05 06 07
      08 09 0A 0B 0C 0D 0E 0F
      10 11 12 13 14 15 16 17
      18 19 1A 1B 1C 1D 1E 1F

   02 20
      2B 01 02 03 04 05 06 07
      08 09 0A 0B 0C 0D 0E 0F
      10 11 12 13 14 15 16 17
      18 19 1A 1B 1C 1D 1E 1F
```

整个 DER 71 字节：

```text
SEQUENCE Tag      1
SEQUENCE Length   1
SEQUENCE Value   69
-------------------
总计             71
```

### 二、现在真正按照 Parser 的视角解析

假设：

```text
p
↓
30 45 02 21 00 A1 ...
```

**第一步：**

```text
Tag = 30

# 解析 Tag = 00 | 1 | 10000
# 得到：
    Universal
    Constructed
    Tag Number 16
```

所以：SEQUENCE

**第二步读取 Length**

下一 Byte：

```text
45
```

最高 bit 0，所以是：**Short Form**

长度：*0x45 = 69*

于是我们立即知道：

> 后面的 69 Byte 全部属于这个 SEQUENCE。
>

这一刻就确定了：

```text
SEQUENCE 的边界
```

**第三步 SEQUENCE Value**

现在：

```text
p
↓
02 21 00 A1 ...
```

看到：

```text
02

即：Tag = INTEGER
```

再读：

```text
21

即：Length = 0x21 = 33
```

于是接下来：

```text
33 Byte
```

全部属于第一个 INTEGER。

**第四步读出第一个 INTEGER**

Value：

```text
00 A1 01 02 ... 1F
```

33 Byte。

看到最前面：

```text
00
```

这时候不要马上认为：r 本身就是 33 Byte。

结合 INTEGER 规则判断。

后一个 Byte：**A1** ，最高位为 1

因此这个：

```text
00
```

是：**必须存在的正数符号保护字节。**

如果你最终要转换回固定长度 32 Byte 的 raw r：

```text
00 A1 01 ... 1F
```

应该恢复为：

```text
A1 01 ... 1F
```

正好 32 Byte。

**第五步指针移动到第二个字段**

第一个 INTEGER 总长度：

```text
1 + 1 + 33 = 35
```

所以解析后指针移动：

```text
p += 35
```

来到：

```text
02 20 2B 01 ...
^
p
```

**第六步第二个 INTEGER**

```text
02 20

表示：INTEGER
     Length = 32
```

Value：

```text
2B 01 ... 1F
```

因为：*2B \< 80，最高 bit = 0，所以没有前导 ***00*** *

直接就是原始 s。

**最终还原**

所以这 71 Byte DER：

```text
30 45 ...
```

最终恢复：

```text
Signature
├─ r = 32 Byte
└─ s = 32 Byte
```

也就是 raw signature：

```text
r || s
```

仍然是：64 Byte

### 三、为什么不能用 memcpy 做 DER ↔ Raw

你可能现在已经发现：

DER：

```text
71 Byte
```

Raw：

```text
64 Byte
```

因为 DER 里面还包含：

```text
30          SEQUENCE Tag
45          SEQUENCE Length

02          INTEGER Tag
21          INTEGER Length
00          Sign Protection

02          INTEGER Tag
20          INTEGER Length
```

这些都是：**编码结构**，不是 r / s 的实际值。

**Raw Signature 和 DER Signature**

这两个概念一定要区分。

**Raw**

```text
r || s
```

对于 256-bit 曲线：

```text
32 Byte r
+
32 Byte s
=
64 Byte
```

特点：

```text
固定长度
无 Tag
无 Length
无 INTEGER 包装
```

**DER**

```text
SEQUENCE
├─ INTEGER r
└─ INTEGER s
```

特点：

```text
带 Tag
带 Length
INTEGER 有符号规则
总长度可能变化
```

所以：

```text
Raw Signature
≠
DER Signature
```

这是 SM2 / ECDSA 工具开发里非常常见的问题。

### 四、DER 签名可能更短

如果一个 32 Byte 正整数最前面本来有很多：

```text
00
```

DER 为了满足：**最短 INTEGER 编码**

还会去掉不必要的前导 `00`。

例如 raw 固定宽度：

```text
00 00 12 34 ...
```

在 raw 格式里要求固定宽度：

```text
必须保留到 32 Byte
```

但 DER INTEGER 可能只保留：

```text
12 34 ...
```

只要：

```text
12
```

最高位是 0。

所以 DER 转 Raw 时还有另一个任务：

> **必要时在左侧补 0，恢复固定 32 Byte。**
>

### 五、因此 DER → Raw 的核心逻辑

以 32 Byte r/s 为例：

```text
DER INTEGER
     ↓
去除编码层必要的符号 00
     ↓
检查数值长度 <= 32
     ↓
左侧补 00
     ↓
恢复 32 Byte
```

比如 DER INTEGER Value：

```text
12 34
```

转换成 raw 32 Byte：

```text
00 00 00 ... 00 12 34
```

不是把：**12 34 **直接放到 raw 前面。

因为整数通常按：**Big Endian **表达。

要左侧补零。

**Raw → DER 则反过来**

对于固定 32 Byte r：

```text
00 00 00 A1 ...
```

首先去掉不必要的前导：

```text
00
```

直到得到最短整数表达。

但如果去完之后第一个有效 Byte：

```text
>= 0x80
```

又必须添加一个：00  保护正数。

可以理解为：

```text
Fixed-width Raw Integer
        ↓
去掉固定宽度补零
        ↓
检查最高有效 bit
        ↓
必要时加 00
        ↓
DER INTEGER
```

**为什么密码工具特别容易碰到这个？**

因为你在做：

```text
SM2
```

时，很容易在不同接口之间碰到：

```text
Raw r||s
DER SEQUENCE(INTEGER r, INTEGER s)
Hex
Base64
```

甚至调用 OpenSSL 或其他库时：

> 一个接口要 DER，一个接口返回 Raw。
>

如果没有搞清楚这一章，很容易出现：

```text
签名明明是对的
但验签一直失败
```

原因可能不是密码算法错了，而只是：**编码格式不一致。**

### 六、现在再往上一层：Hex 是什么？

我们刚才写：

```text
30 45 02 21 ...
```

这是什么？它不是 DER 的另一种“格式”。

真正 DER 数据在内存里是：

```text
Byte
Byte
Byte
...
```

例如第一个 Byte：

```text
0011 0000
```

我们为了方便人阅读，写成：30

因此：

```text
30 45 02 21 ...
```

只是：DER Binary 的 Hex Dump / Hex 表示。

关系：

```text
DER Binary
      ↓
用 Hex 打印
      ↓
30 45 02 21 ...
```

数据本质没变。

**Hex String 又要区分**

假设真正 Binary：

```text
30 45
```

内存是：

```text
0x30 0x45
```

只占：

```text
2 Byte
```

但如果字符串：

```text
"3045"
```

内存是 ASCII：

```text
33 30 34 35
```

占：

```text
4 Byte
```

所以：**DER Binary 和 DER Hex String 完全不是一回事**

这是第 4 章学过的，现在和 DER 接起来了。

### 七、那么 Base64 在哪一层？

我们的完整 DER Binary：

```text
71 Byte
```

可以拿去做：**Base64 Encode**

得到文本：

```text
MEUCIQChAQIDBAUGBwgJCgsMDQ4PEBESExQVFhcYGRobHB0eHwIgKwECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=
```

注意：

> Base64 并没有理解什么叫 SEQUENCE、INTEGER、r、s。
>

它根本不懂 ASN.1。

Base64 只做：

```text
Binary
↓
Printable ASCII
```

所以：

```text
DER
```

是结构编码。

而：

```text
Base64
```

是二进制到文本的表示转换。

层次完全不同。

### 八、如果再加 PEM 呢？

假设我们人为给这种数据做一个文本包装：

```text
-----BEGIN SIGNATURE-----
MEUCIQChAQIDBAUGBwgJCgsMDQ4PEBESExQVFhcYGRobHB0eHwIgKwECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8=
-----END SIGNATURE-----
```

这就是：

```text
PEM 风格包装
```

核心关系：

```text
DER Binary
      ↓
Base64
      ↓
BEGIN / END 文本包装
      ↓
PEM
```

注意真实标准使用什么 PEM Label，要由对应标准规定；这里主要是在说明层次关系。

**所以解 PEM 的顺序是什么？**

以后你拿到：

```text
-----BEGIN ...-----
MIIB...
-----END ...-----
```

不要直接想着：**“ASN.1 怎么解析这个字符串？”**

正确流程是：

```text
PEM
 ↓
去掉 Header / Footer
 ↓
取得 Base64 文本
 ↓
Base64 Decode
 ↓
DER Binary
 ↓
DER Parser
 ↓
ASN.1 Structure
```

这条链必须掌握。

反方向，如果程序内部已经有：

```text
r
s
```

那么：

```text
程序整数 r/s
      ↓
ASN.1 Signature 结构
      ↓
DER Encode
      ↓
Binary
      ↓
Base64 Encode
      ↓
添加 PEM Header/Footer
      ↓
PEM Text
```

### 九、现在终于能把这些词正确放到不同层级了

这是这一整套课程非常重要的一张图：

```text
┌─────────────────────────────┐
│          Meaning            │
│       r / s / PublicKey     │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│            ASN.1            │
│     SEQUENCE / INTEGER      │
│      数据结构 / Schema      │
└──────────────┬──────────────┘
               │ DER Encode
               ▼
┌─────────────────────────────┐
│        DER Binary           │
│ 30 45 02 21 00 A1 ...      │
└──────────┬──────────┬───────┘
           │          │
           │ Hex      │ Base64
           ▼          ▼
       Hex String   Base64 Text
                      │
                      ▼
                     PEM
```

**特别注意：ASN.1 和 PEM 之间隔了好几层**

以后有人说：**“这个 PEM 是 ASN.1 格式。”**

你应该能理解对方大概率想表达什么，但你自己的思维要更严谨：

```text
PEM
└─ Base64
   └─ DER
      └─ 某个 ASN.1 Structure
```

所以：PEM 只是最外面的文本包装。

### 十、一个实际的排错思路

以后你遇到一段数据：

```text
MEUCIQ...
```

别人说：“这是 SM2 签名。”

不要马上拿去验签。

先问：

```text
它现在是什么表示？
```

例如可能是：

- 情况 A：Raw 64 Byte Binary
- 情况 B：64 Byte Raw 的 Hex String
- 情况 C：DER Binary
- 情况 D：DER 的 Hex String
- 情况 E：DER 的 Base64

这些：

> 逻辑上都可能代表同一个 SM2 签名。
>

但 Byte 完全不同。

**这就是“数据表达”为什么值得专门学**

我们最开始开这门课的时候，你问过一个非常关键的问题：

> 理解 bin、Hex、PEM、ASN.1、Base64 等，
>
> 对不同设备、不同链路之间的数据交流是不是有必要？
>

到这里答案已经非常明显。

同样一个逻辑签名：

```text
r
s
```

可能层层变成：

```text
r / s
     ↓
    Raw
     ↓
    DER
     ↓
    Hex

# 或者：

    r / s
     ↓
    DER
     ↓
    Base64
     ↓
    PEM
```

如果不知道当前在哪一层：

> 你连“手里拿到的到底是什么”都可能判断错。
>

### 十一、工业级 DER Parser 需要做更多检查

不仅要：

```text
能读出来
```

还应该检查：

```text
Tag 是否正确
Length 是否越界
Length 是否使用规范形式
INTEGER 是否为空
INTEGER 是否存在不必要前导 00
INTEGER 是否错误表示负数
SEQUENCE 内部是否刚好解析完
是否存在额外尾随 Byte
```

尤其做：

```text
密码学
安全协议
证书
```

时：**宽松解析并不总是好事。**

**例如这个 INTEGER**

```text
02 02 00 01
```

逻辑上：

```text
1
```

但是 DER 中：00 是多余的

因为：01 最高位本来就是 0

正确 DER：

```text
02 01 01
```

所以严格 DER parser 应该拒绝：

```text
02 02 00 01
```

这就是：**Canonical Encoding**

真正落到 Byte 层面的含义。

### 十二、建立一个“拆数据”的固定流程

以后拿到未知密码数据，按照这五步走。

**第一步：先判断当前表示层**

```text
Binary？
Hex String？
Base64？
PEM？
```

**第二步：恢复原始 Binary**

例如：

```text
PEM
→ Base64 Decode
→ Binary
```

或者：

```text
Hex String
→ Hex Decode
→ Binary
```

**第三步：判断 Binary 格式**

```text
Raw？
DER？
其他协议？
```

**第四步：如果 DER，按 TLV 解析**

```text
Tag
→ Length
→ Value
→ Nested TLV
```

**第五步：恢复逻辑字段**

例如：

```text
SEQUENCE
├─ INTEGER r
└─ INTEGER s
```

最终得到：

```text
r
s
```

### 十三、第 6 章最核心的完整链路

现在整章可以浓缩成：

```text
程序里的数据
      │
      ▼
Serialization
      │
      ├─ Fixed Layout
      │
      └─ TLV
           │
           ▼
         ASN.1
      描述结构
           │
           ▼
          DER
    规范化 TLV 编码
           │
           ▼
      Binary Bytes
       /        \
      /          \
    Hex        Base64
                 │
                 ▼
                PEM
```

**再把第 5 章和第 6 章连接起来**

假设真正设备协议：

```text
AA 55 | Length | Command | Payload | CRC
```

其中 Payload 放一个 DER 签名。

那么整体可能是：

```text
Frame
│
├─ Header
├─ Length
├─ Command
│
├─ Payload
│    │
│    └─ DER
│        │
│        └─ SEQUENCE
│            ├─ INTEGER r
│            └─ INTEGER s
│
└─ CRC
```

这时候协议存在多个层：

```text
UART/TCP Byte Stream
        ↓
Frame
        ↓
Payload
        ↓
DER
        ↓
ASN.1 Fields
        ↓
r / s
```

这就是实际工程里真正的数据分层。

### 十四、千万不要跨层解析

例如 Frame Parser 不应该去关心：

```text
r
s
```

它只应该关心：

```text
Frame 完不完整？
CRC 对不对？
Payload 在哪里？
```

DER Parser 也不应该关心：

```text
UART 有没有粘包
```

它只处理：

```text
一段完整 DER buffer
```

再往上的业务层才关心：

```text
这个 INTEGER 是 r 还是 s
```

正确分层：

```text
Stream Layer
    ↓
Frame Layer
    ↓
Serialization Layer
    ↓
Business Layer
```

这和我们第 5 章、第 6 章的安排完全一致。

如果 71 Byte DER 做 Base64：

```text
MEUCIQCh...
```

Base64 字符：

```text
M
E
U
C
...
```

是不是 ASN.1 Tag？

当然不是，必须先：

```text
Base64 Decode
```

恢复：

```text
30 45 02 21 ...
```

才能进入 DER / ASN.1 层。

### 第 6 章总结

第 6 章 5 个课时到这里正式结束。

我们完成了：

### 第 1 课：Serialization

理解：

```text
程序结构
≠
Wire Format
```

以及：

```text
Serialize / Deserialize
```

### 第 2 课：TLV

理解：

```text
Tag    = 我是谁
Length = 我多长
Value  = 我的数据
```

并理解：

```text
可选字段
扩展字段
未知字段
Nested TLV
```

### 第 3 课：ASN.1 / BER / DER

明确：

```text
ASN.1
=
描述数据结构

BER / DER
=
编码规则
```

以及：

```text
DER ⊂ BER
```

DER 强调：

```text
Canonical Encoding
```

### 第 4 课：DER Byte 规则

学会：

```text
Tag
Length
Value
```

以及：

```text
INTEGER
BIT STRING
OCTET STRING
SEQUENCE
```

尤其掌握：

```text
Short Form
Long Form Length
INTEGER Sign
BIT STRING unused bits
```

### 第 5 课：真实 DER 实战

我们已经能够从：

```text
30 45 02 21 00 ...
```

手工恢复：

```text
SEQUENCE
├─ INTEGER r
└─ INTEGER s
```

并进一步串通：

```text
ASN.1
 ↓
DER
 ↓
Binary
 ↓
Base64
 ↓
PEM
```

### 到这里，你对最初那些概念应该有了完全不同的认识

最开始可能是：

```text
bin
Hex
Base64
PEM
ASN.1
DER

好像全是各种“格式”
```

现在应该变成：

```text
数据意义
   │
   ▼
ASN.1
结构描述
   │
   ▼
DER
结构编码
   │
   ▼
Binary
真实 Byte
   │
   ├── Hex
   │   人类查看二进制
   │
   └── Base64
       Binary → Text
          │
          ▼
         PEM
       文本包装
```
