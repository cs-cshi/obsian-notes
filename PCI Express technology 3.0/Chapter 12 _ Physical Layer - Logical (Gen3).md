Gen3 在速率未翻倍（5GT/s -> 8GT/s）的情况下，实现了带宽的翻倍（500MB/s -> 984.6MB/s），相比PCIe Gen1、Gen2，Gen3：
- 新的编码模型：采用了更为高效的 128b/130 数据编码，每个时钟周期传输128-bit有效数据。
- 更复杂信号均衡模型：速率提升，Gen1 2.5 GT/s，Gen2 5 GT/s，Gen3 8 GT/s。随着频率提升，需要更强的信号完整性技术，更稳健的信号补偿机制。

![pcie rate.png](./images/pcie-rate.png)
# 1. Introduction to Gen3
PCIe 链路进入训练状态时（如复位后），会先以 Gen1 的速度开始，以实现向后兼容。当双方设备能以更高速度时，链路将立即转换至 Recovery 状态，将速率改为链路双方共同支持的最高速度。
PCIe 升级到 Gen3 的主要动机是实现带宽提升一倍。这可以通过信号速率提升一倍（5GT/s -> 10GT/s）实现，但存在如下问题：
- 对于更高频率，功率要求更高，也需要更复杂的均衡逻辑（equalization）来维持更高速率下的信号完整性。实际上， PCISIG 指出，均衡逻辑的高功率需求是维持频率尽可能低的一个重要原因。
- 电路板材的材料和设计要求更高。
- 频率提升越高对基础设施（电路板、连接器等）的要求也会越高，难以使用已有基础设施降低成本。
## 1.1 New Encoding Model
对于 8b/10b 模型，传输 8-bit 有效数据，实际必须传输 10-bit，这样会产生 20% 的传输开销。Gen3 使用 128b/130b 模型，可以极大程度降低传输开销（存在 2/130 的开销）。此时 Gen3 信号速率只需提升到 8GT/s，实际带宽能翻倍提升到 1GB/s。此外，8b/10b 编码中使用控制字符标识数据包边界（如下图所示），接收端通过识别这些控制字符，获取接收的数据类型。
<center>Figure 12-1: 8b/10b Lane Encoding</center>
<center></center>
![](./images/8b_10b-Lane-Encoding.png)

128/130b 编码将数据包分成一个个包含 16 字节的块（block），每个块开头增加一个 2-bit 的同步字段（Sync Header），用来表示当前块是数据（10b）还是有序集ordered set（01b）。同步字段和有序集必须在所有 lane 上同时传输，链路训练中实现 lane 之间的同步。
<center>Figure 12-2: 128b/130b Block Encoding</center>
![](./images/128b_130b-Block-Encoding.png)
## 1.2 Sophisticated Signal Equalization
Gen3 的第二项变化是在物理层的电气模块。Gen1、Gen2 采用预先固定的 Tx 去加重（de-emphasis），以达到良好的信号质量，但速率在 5GT/s 以上时会导致严重的信号完整性问题。Gen3 使用额外的均衡开销，设计在 PHY 的发送端和接收端链路上。
# 2. Encoding for 8.0 GT/s
128b/130b 在整个链路范围封装数据报文，在单个 lane 上对数据分块编码。
## 2.1 Lane-Level Encoding
每个 lane 上的块先是 2-bit 同步头（Sync Header），后面是 16B 的信息，共 130bit。同步头用来标识发送的是数据块（10b）还是有序集（01b）。需要注意的是，链路中先传输的是最低有效位，因此图中同步头是 01（小端），表示数据块 10b（大端）。
<center>Figure 12-3: Sync Header Data Block Example</center>
![](./images/Sync-Header-Data-Block-Example.png)
## 2.2 Block Alignment
与 Gen1、Gen2 一样，接收端首先需要进行位锁定（Bit Lock），然后 Gen3 会进行块锁定（Block Alignment Lock），这需要接收端能从比特流中知晓划分块边界的同步头（仅通过 01h 和 10 h 无法判断第一个块的开始）。发送端通过发送由 00h、FFh 交替组成的 EIEOS 来建立此边界。在 Gen3 中 EIEOS 不仅用于退出电气空闲，而用于建立块对齐机制，同步头会紧随在 EIEOS 前后。
Figure 12-4: Gen3 Mode EIEOS Symbol Pattern
![](./images/Gen3-Mode-EIEOS-Symbol-Pattern.png)
## 2.3 Ordered Set Blocks
与 Gen1、Gen2 一样，Ordered Sets 是用于管理 lane protocol 正常运行（可能涉及同步、握手、较准等），以保证数据在通道上的可靠传输。
当发送有序集块时，它必须同时出现在所有的通道上，并且除了 SOS（SKP 有序集）外都是由 16 bytes 组成。SOS 用于时钟补偿，以 4 bytes 为一组的形式添加/删除 SKP Symbol，其长度可以为 4、8、16、20、24 等。
<center>Figure 12-5: Gen3 x1 Ordered Set Block Example</center>
![Figure 12-5: Gen3 x1 Ordered Set Block Example](./images/12-5.png)

Ordered Set 块与数据块只是同步头值相反。Gen3 定义了 7 个有序集（比 Gen1/2 多一个 SDS）
- SOS：Skip Ordered Set，用于时钟补偿
- EIOS：Electrical Idle Ordered Set，用于进入电气空闲
- EIEOS：Electrical Idle Exit Ordered Set
   - 退出电气空闲
   - 块对齐边界
- TS1：Training Sequence 1 Ordered Set
- TS2：Training Sequence 2 Ordered Set
- FTS：Fast Training Sequence Ordered Set，用于快速建立退出电气空闲时的位锁定和时钟
- SDS：Start of Data Stream Ordered Set
<center>Figure 12-6: Gen3 FTS Ordered Set Example</center>
![Figure 12-6: Gen3 FTS Ordered Set Example](./images/12-6.png)

上图展示了 Gen3 FTS 有序集的组成，其中 Sync Header 01b 表示当前块为有序集块，并通过块中第一个 Symbol 识别为 FTS 类型，图右侧列出了其他类型有序集第一个 Symbol 值。
## 2.4 Data Stream and Data Blocks
发送端通过发送 SDS 有序集，使链路处于 L0 状态开始传输数据流。数据流以 EDS 标识（Token）结束，其间可以传输多个数据块（除非出现错误）。EDS Token 总是位于有序集前最后一个数据块的后 4 个 Symbol。注意 SKIP 是例外，因为 SKIP 后如果还是数据块意味这数据流没有停止。
## 2.5 Data Block Frame Construction
数据块由 TLPs、DLLP 和 Token 组成，Token 用于标识当前块中类型/状态。在一个数据块中可能会用到五种类型的 Token，下节会介绍。图 12-7 展示了一个单通道 TLP 传输组成的数据块示例。
<center>Figure 12-7: Gen3 x1 Frame Construction Example</center>
![Figure 12-7: Gen3 x1 Frame Construction Example](./images/12-7.png)
### Framing Tokens
共有 5 种 Token（帧标识）：
- STP：表明是 TLP 报文（DW 双字，4B）
- SDP：表明是 DLLP 报文（单字，2B）
- IDL：逻辑空闲（1B），没有 TLPs/DLLPs 传输时，发送 0 字节
- EDS（End of Data Stream）：数据流正常结束（DW，4B）
   - 该 Token 后至少有一个有序集。如果 EDS 后是 SOS + 数据块，不会结束数据流，否则结束。
- EDB：使 TLP 无效数据包（DW，4B）（注意没有 END，当没有 EDB 时默认良好）

数据块的内容因链路当前的活动（activity）而异：
- IDLs：当没有数据包传送时，Data Blocks 只包含 IDL（Token）
- TLPs：根据链路宽度，可以在一个 Data Block 中发送一个或多个 TLP
- DLLP：一个或多个 DLLP 可在 Data Block 中发送
- 以上可以组合在一个 Data Block 中发送
<center>Figure 12-8: Gen3 Frame Token Examples (little-endian)</center>
![Figure 12-8: Gen3 Frame Token Examples (little-endian)](./images/12-8.png)
注意 Spec 中是大端 big-endian
### Packets
以 12-7 图为例，STP/SDP 标识数据包的开始：
- TLPs：STP Token 以全 1 的半字节（nibble）开始。随后是 11-bit 的长度字段（$2^{11} = 2k$双字，8KB TLP），用于对 TLP 双字计数（Token, header, optional data payload, option digest, LCRC），便于接收端识别 TLP 结束位置。Length 字段非常重要，因此有一个检验 length 字段的 4-bit 的 CRC 字段以及检验 length 字段和 CRC 字段的偶数奇偶校验位（Parity bit），两者能检测 3 bit 错。
   - 正常结束的 TLPs 报文没有 END，如果存在损坏会在末尾添加 EDB，作为 TLP 的扩展，Length 字段不会记录。此外损坏 TLP 还会反转 LCRC 值。
- DLLPs：SDP Token 没有长度字段，因为 DLLP 固定 8 bytes：2B Token + 4B payload + 2B LCRC，不需要标识良好结束的 END。
### Transmitter Framing Requirements
数据流从 SDS 之后的第一个 Symbol 开始，它可能包含由 Token、TLPs 和 DLLPs 组成的数据块。数据流以有序集（除 SOS）之前的最后一个 Symbol 结束，或者检测到错误结束。除 SOS 外，数据流期间不能发送任何有序集。
当帧错误发生时，该错误会被接收方视为接收端错误，会向上层报告，同时通过将 LTSSM 从 L0 引导到 Recovery state 启动恢复流程。此时接收端会停止处理当前的数据流，直到检测到 SDS 有序集时开始处理一个新的数据流。
> PCIe 使用一种称为“通道聚合”的技术，其中多条称为“lane”的物理通道同时工作，以增加总线带宽和数据传输速度。数据帧中的信息可以同时通过多条 lane 传输，这样就能够提高总线的传输速度和带宽。每个 lane 负责传输部分数据。

发送一个 TLP：
- STP token 后紧跟 TLP 的全部内容，即使 TLP 数据已无效。
- EDB token 出现在无效 TLP 的最后一个双字中，不包含在 TLP Length 字段计数。
发送一个 DLLP：
- SDP token 后紧跟 DLLP 全部内容
发送 SOS（SKIP Ordered Set）
- 当前数据流（可能被SOS截断）最后一个数据块最后一个双字发送 EDS Token
- SOS 作为下一个有序集
- SOS 之后立即发送新的数据块，数据流恢复。
- SOS 不能连续发送（Gen 1/2 可以），必须间隔 TLP、DLLP or IDL。
结束数据流，需要在数据块最后一个双字发送 EDS token，然后使用 EIOS 进入低功耗状态，或者在所有其他情况下使用 EIEOS。
当链路上没有 TLP、DLLP 或者其他帧令牌时，在所有通道上发送 IDL token。

多 lanes：
- IDL token 后，下一个 TLP/DLLP 必须位于 lane0。
- IDL token 可用于填充 Symbol Time 数据。如 x8 链路在 lane3 结束 TLP，但新的发送数据没准备好，此时使用 IDL 填充后续 lane。
- 数据包以 DW 对齐，即 4 字节，因此数据包可在以 4 对齐的 lane 号开始，前一个 lane 结束。
### Receiver Framing Requirements
当期望帧标识（Framing Token）时，其他任何数据（Symbols）都被认为帧错误（Framing Errors）。
下面所有 error checks 可选，并彼此相互独立。
接收到 STP 时：
- 检查帧的 CRC 和 Parity 奇偶校验字段，有错视为帧错误（Framing Error）
- TLP 最后一个 DW 之后的 Symbols 是下一个要处理的 Token，接收端检查是否是 EDB token，若是标识 TLP 无效。
- Length 字段若为 0 表示帧错误。（可选）
- 同一个 Symbol Time 内若有多个 STP Token 到达视为真错误。（可选）

接收到 EDB 时：
- 检测到第一个 EDB 符号，或者在接收到它的任何剩余字节之后，接收端必须立即通知链路层。
- EDB token 只能出现在 TLP 后，任何其他时间地方视为帧错误
- If any Symbols in the Token are not EDBs, the result is a Framing Error
- EDB 后第一个 Symbol 是下一个需要处理的 Token。

当 EDS Token 作为数据块最后一个 DW 被接收时：
- 接收端停止处理当前数据流
- 之后下一个 Symbol 只能是 SKP, EIOS, or EIEOS Ordered Set，其他 Ordered set 均视为帧错误。
- EDS 后是 SKP Ordered Set，接收端将 EDS 后第一个 Symbol 作为数据块并恢复数据流。

接收到 SDP Token 时：
- DLLP 之后是下一个要处理的 Token

接收到 IDL Token 时：
- IDL token 之后的 token 在任何 DW 对齐 lane 上开始。如 lane0、4、8、12 ...
- 同一 Symbol Time 可接收的是另一个 IDL 或 EDS。

接收端在处理数据流中可能出现的帧错误：
- Ordered Set 紧跟在 SDS 后
- 非法同步头（Sync Header, 11b or 00b）


### Receiver Framing Requirements
### Recovery from Framing Errors
发送端成帧
## 3. Gen3 Physical Layer Transmit Logic
## 4. Gen3 Physical Layer Receive Logic
## 5. Notes Regarding Loopback with 128b/130b
