# LX06 多音箱音频同步（mdplay）

供后续 agent 使用。本文只描述 **LX06 1.74.10** 明文 rootfs 里三份闭源库的静态反编译结果，不是运行时抓包，也不是上游 Snapcast 源码的复述。

伪代码来自 REA CLI `6.8.0` + Ghidra `12.1.4`。Ghidra 输出不是原始源码。`Stream::getPlayerChunk` 里的 `VectorShiftLeft` / `VectorAdd` 链是 64 位时长运算反编译失败，**不要把那条链当成乘法公式**。

## 产物

| 文件 | SHA-256 | 角色 |
|---|---|---|
| `lx06-fw/rootfs/usr/lib/libmdplay.so` | `f3c1b2983cc0dad9aeeb0c7effb6002e59e27092b80a5b0e404cec7cc103e04d` | 对时、时间戳、缓冲、变速、ALSA 播出 |
| `lx06-fw/rootfs/usr/lib/libiotdcm_mdplay.so` | `0572fac5e89b875319fabee82ea2aa3042ce85158305a6c22368f48f9c98b7e6` | `/usr/bin/mdplay` 实际使用的组网与发送 |
| `lx06-fw/rootfs/usr/lib/libiotdcm.so` | `3bf6d690a7e7da35310d9bc65c8790b509ba6d6069932fe964a10ec539f53feb` | 另一份 star-trek iotdcm，含云端 Ping，不算 DAC 时钟 |

三份都是 ARM 32 位、已剥离符号的 ELF。`libmdplay.so` 全量反编译证据：`ev_4eb7b6df57ab7613b64989a40a7a9268b2394b9641a8ab96cf156900e792d753`（1211 个函数）。

解包步骤见 [ota-download-unpack.md](./ota-download-unpack.md)。

## 结论（先看这个）

多音箱对齐靠的是 **服务器墙上时钟 + 每块音频的服务器时间戳 + 客户端估计的时钟偏移**，不是 iotdcm 的 RTT，也不是云端 `if_need_sync_time`。

1. 组关系（谁是服务器、谁加入、立体声左右）由 iotdcm 的 `MultiroomGroup` / `StereoGroup` 交给 `/usr/bin/mdplay`。库内用 `is_Server` 区分角色。
2. 服务器给每一块 PCM 打上自己的 `gettimeofday` 时间戳，把同一块发给组里每个 session。
3. 客户端周期发送类型 4 的 Time 消息。服务器把「收到时刻 − 客户端发送时刻」写进载荷并回包。
4. 客户端计算 `偏移 = (服务器差 − 客户端差) / 2`，单位微秒，最多 20 个样本做中值/中间三点平均。
5. 播放时：客户端用 `本地时间 + 偏移` 当作服务器时间；服务器直接用本地时间。再减去声卡里已经排队的样本时间，决定这一块现在该播、该丢还是该等。
6. 剩余误差用 ±0.01% 采样率微调；误差大时硬改播放偏移。

## 库边界（不要算错地方）

`libmdplay.so` 不自己建 UDP。序列化后的字节通过：

- 客户端：`g_send_client_cb(buf, len+1, flag, g_send_client_user)`
- 服务器：`g_send_server_cb(...)`

`flag`：类型 4（Time）为 1，类型 5（Hello）为 2，其余（含音频）为 0。回调由 `mdplay_client_init` / `mdplay_server_init` 注入。`/usr/bin/mdplay` 再调用 `iotdcm_send_data_single`。

`libiotdcm_mdplay.so` 的 `iotdcm_send_data_single` 把缓冲区原样交给内部发送函数（`FUN_0006ae2c`），**不写时间戳**。同库的 XMD Ping/Pong、FEC、丢包恢复是传输可靠性，量的是链路，不决定样本在哪一微秒播出。

`libiotdcm.so` 的 `if_need_sync_time` 在 `SendPingPacket`（源码路径字符串 `star-trek_special/lib/src/user.cpp`）。日志同时打出 `last_pign_ts`、`ping_req_time_interval`、`mds_config_version`、`members_version_`。这是云端成员保活。证据 `ev_313ca6d7d850fa60fefbe5e7ac23245a3ad021a784392393295d55113d369c04`，字符串证据 `ev_a1e1685c1e3b72cc5ff1a4f37001110b6ab30c3c17700f5b55ceac977f13ac3b`。

## 角色

| `is_Server` | 时钟 | 播放比较 |
|---|---|---|
| 非 0（服务器） | 本机 `gettimeofday` / `system_clock` 就是基准 | 块时间戳对本地时间，不加 `TimeProvider` 偏移 |
| 0（客户端） | 用 Time 回包估计「服务器时钟 − 本机时钟」 | 比较前把本地时间加上该偏移 |

立体声左右声道是播出前的样本拷贝，不改变时钟：

- `gmdplay_channel == 0`：16-bit 立体声帧把左声道（字节 0、1）复制到右声道（字节 2、3）
- `gmdplay_channel == 1`：把右声道复制到左声道

多房间是各机播同一路并靠时钟对齐；立体声是在此之上只留一侧。`mdplay_channel_config` 设置该全局量。

## 线格式

小端。`msg::BaseMessage::serialize` 的写出顺序（也是 `read` 的读入顺序）：

| 字段 | 宽度 | 对象内偏移 |
|---|---|---|
| type | uint16 | +0x04 |
| id | uint16 | +0x06 |
| refersTo | uint16 | +0x08 |
| sent.sec | int32 | +0x14 |
| sent.usec | int32 | +0x18 |
| received.sec | int32 | +0x0c |
| received.usec | int32 | +0x10 |
| size | uint32 | +0x1c |
| payload | `size` 字节 | Time 从 +0x20 起 |

头长 **26**（`0x1a`）字节。接收端先读 26 字节头，再按 `size` 拷载荷。`size > 1000000` 服务器拒绝。

观察到的 type（`Controller::onMessageReceived` 与服务器 `type > 6` 的拒绝）：

| type | 消息 | 载荷 |
|---|---|---|
| 1 | CodecHeader | 编码名与参数。客户端据此建 decoder、`Stream`、`AlsaPlayer` |
| 2 | WireChunk / PcmChunk | 见下 |
| 3 | ServerSettings | JSON：`bufferMs`、`latency`、`volume`、`muted` |
| 4 | Time | 8 字节：`int32 sec`、`int32 usec`（一个 timeval **差**，不是绝对时刻）。`getSize()` 返回 8 |
| 5 | Hello | 客户端加入。`Controller::worker` 首先发送 |

WireChunk 载荷：`int32 sec`、`int32 usec`、`int32 nbytes`、随后 `nbytes` 字节。`getSize()` = `nbytes + 12`。`sec/usec` 是服务器采集该块时的 `gettimeofday`。`PcmChunk::start` 使用：

```text
块内位置的绝对微秒 = sec * 1000000 + usec + (已索引样本 / 采样率) * 1000000
```

## 对时算法

### 谁发送

只在客户端（`is_Server == 0`）。

`Controller::sendTimeSyncMessage(min_interval_ms)` 用 `CLOCK_MONOTONIC` 的毫秒做节流：距上次发送不足 `min_interval_ms` 则直接返回。发出去的 Time 消息 type=4，`refersTo=0`，sent 与 received 都填当前 `gettimeofday`，载荷里也先放一个当前 timeval（服务器会改掉载荷）。

三条触发：

1. **启动。** `Controller::worker` 发 Hello 后，最多 20 轮请求/应答。每轮拿到回包就 `TimeProvider::setDiff`，然后 `nanosleep` 100 微秒。
2. **有数据流时。** `onMessageReceived` 在消息不是类型 4、且本机不是服务器时调用 `sendTimeSyncMessage(1000)`，即播放期间大约最多每秒一次。
3. **空闲节拍。** 同一 worker 在 CodecHeader 已收到之后，每 10 次 100 毫秒睡眠调用 `sendTimeSyncMessage(5000)`，所以没有其它消息时大约最多 5 秒一次。CodecHeader 还没到会重发 Hello。

`TimeReSync_thread`（`mdplay_client_create` 里 `pthread_create`）会在 `timeReSync_sem` 上 `sem_wait`。唤醒后若距上次突发已超过 6 秒，则连发 12 次、间隔 200 毫秒的 `sendTimeSyncMessage(200)`。`mdplay_client_init` 只 `sem_init(..., 0)`。这三份库和已搜索的 rootfs 里 **没有 `sem_post`**。不要把这条线程写成固件里会自动发生的重同步。

### 服务器如何填载荷

`mdplayTimeReceive_server`：type == 4 时

- `refersTo = id`
- `payload = received - sent`（两个 timeval 相减）
- usec 为负：usec += 1000000，sec -= 1
- `StreamSession::sendAsync` 回给对应客户端 index

`received` 是到达本机后盖上的本地时刻，`sent` 仍是客户端发出时写入的时刻。二者时钟不同，所以这个差 = 时钟偏移 + 单向时延，不是纯 RTT。

### 客户端如何得到偏移

本地差用同一对端点：`客户端收到回包的时刻 − 当初的 sent`。这是同一台机器上的两个 `gettimeofday`，数值上接近 RTT（含服务器处理）。

`TimeProvider::setDiff` 与 `setDiff2` 的换算相同。参数是两个 timeval `(sec1, usec1)` 与 `(sec2, usec2)`，前者是回包载荷（服务器差），后者是客户端差：

```text
d_ms_half = (usec1 - usec2) * 0.0005 + (sec1 - sec2) * 500
offset_us = (int64)(d_ms_half * 1000)
          = (usec1 - usec2) / 2 + (sec1 - sec2) * 500000
          = ((服务器差) - (客户端差)) / 2     单位：微秒
```

代数上等于 `(服务器回包时刻 − 客户端收到回包时刻) / 2`，因为两边都减去了同一个客户端 `sent`。这是一次估计，**不是**四时间戳 NTP（没有单独的 T2/T3）。对称时延假设没有在代码里再修正。

样本处理（两函数相同的部分）：

- 队列上限 20（构造函数先写 10，随即改成 `0x14`）
- 满了就丢掉最旧样本
- 排序后：样本数 < 3 取正中间一个；否则取最中间 3 个的整数平均（和除以 3）
- 结果用独占写发布到 `TimeProvider+0x30`（单例对象偏移 48，日志里的 `TimeDiff2Server`）
- 若距上一次成功样本的墙上时间超过 60 秒：打日志 `Last time sync older than a minute`，清空队列，并先把本样本直接发布为当前偏移

只有 `setDiff2`（`mdplayTimeReceive_Client` 在应答没有匹配到 pending request id 时走这里）会维护 `suddenCount`：`abs(新样本微秒 − 当前偏移) > 25000` 则 `suddenCount++`，否则清零。`setDiff`（worker 启动那 20 轮）不改 `suddenCount`。

worker 启动循环里，usec 借位的反编译显示加数是 250000，和另外两处明确的 1000000 不一致，而且在该函数机器码里没有核对到这个立即数。**借位按 1000000 理解；不要采用 250000。**

## 播放对齐

入口：`AlsaPlayer::worker` → `snd_pcm_delay` → `Stream::getPlayerChunk`。

DAC 排队时间：

```text
dac_us = (double)snd_pcm_delay_frames * 20.833333333333332
```

`1000000/48000 = 20.8333…`。这是写死的 48 kHz 帧长，**不是** `1000000/实际采样率`。流不是 48 kHz 时，这个补偿是错的。

客户端比较用的「服务器现在」= 本地 `system_clock` + `TimeProvider+0x30` 的偏移。服务器比较时不加偏移。块的时间来自上一节的 `PcmChunk::start`。同步误差再除以 1000，之后才进入中位数队列、`OutofSync` 日志和统计回调。被除数里有 `* 1000000` 的项，DAC 延迟又以微秒传入，因此除法之后的数最像毫秒。硬校正阈值比日志字符串大三个数量级，见下，单位若要写进新代码请再读 `0x2e71c`。

`Stream::getPlayerChunk`（`0x2e71c`）在播放偏移（对象 +0x10）仍为 0 时：

| 条件（反编译立即数） | 动作 |
|---|---|
| 长窗已满且 `abs(median)` > 1000 | 把播放偏移设为该中位数。日志字符串仍是 `pBuffer->full() && (abs(median_) > 1)` |
| 短窗已满且 `abs(shortMedian)` > 5000 | 同样硬校正。日志字符串仍是 `pShortBuffer->full() && (abs(shortMedian_) > 5)` |
| 否则，短中位数 > 101（`0x65`） | `setRealSampleRate(rate * 0.9999)` 放慢 |
| 否则，短中位数 < -100 | `setRealSampleRate(rate * 1.0001)` 加快 |

日志字符串是 Snapcast 风格的 `> 1` / `> 5`，比较指令用的是 1000 / 5000。**以立即数为准。** 若除法后的单位是毫秒，硬校正要偏差超过 1 秒 / 5 秒；若调用方其实还以微秒在比，则大约是 1 毫秒 / 5 毫秒，才和日志原文的语义一致。这里没有第二份证据消掉这个歧义。

同一除法结果还用于：

- 不在 `[-400, 400]`：计数加一，连续 3 次则打 `OutofSync`，客户端日志带 `TimeDiff2Server` 和 `DacTime`，然后计数清零
- 不在 `[-25000, 25000]`：调用 `g_stat_cb`，键类似 `mdplay_algo_stat`，值为 `{"out_of_sync":1}`

缓冲长度：

- 收到 CodecHeader：`Stream::setBufferLen(1000 - controller.latency)`，`latency` 在 `Controller+0x44`
- 收到 ServerSettings：`setBufferLen(bufferMs - latency)`，两个字段来自 JSON，缺省按 0 填

缓冲吸收抖动；偏移和 ±0.01% 变速负责在缓冲之外把各机拉到同一服务器时间戳。

## 接收入口（daemon 分开喂）

`/usr/bin/mdplay` 导入了四个符号，时间包和数据包不是同一个入口：

- `mdplayTimeReceive_Client` / `mdplayTimeReceive_server`
- `mdplayDataReceive_Client` / `mdplayDataReceive_server`

时间包走 Time 分支并更新 `TimeProvider`。数据包走 `onMessageReceived`：类型 2 解码后 `Stream::addChunk`，类型 1 建播放链并 `AlsaPlayer::start`，类型 3 改缓冲。

## 关键地址（`libmdplay.so`，分析地址）

| 地址 | 符号 |
|---|---|
| `0x3c8d0` / `0x3c9ec` | `msg::Time::doserialize` / `read` |
| `0x33ad0` | `msg::BaseMessage::serialize` |
| `0x392d4` / `0x3934c` | `msg::WireChunk::doserialize` / `read` |
| `0x3a4dc` | `Controller::sendTimeSyncMessage` |
| `0x3b234` | `Controller::worker` |
| `0x3bda4` | `Controller::onMessageReceived` |
| `0x3a928` | `mdplayTimeReceive_Client` |
| `0x48ff8` | `mdplayTimeReceive_server` |
| `0x34ff8` | `TimeProvider::TimeProvider` |
| `0x35bd0` / `0x35670` | `setDiff` / `setDiffToServer` |
| `0x35614` / `0x35054` | `setDiff2` / `setDiffToServer2` |
| `0x2e71c` | `Stream::getPlayerChunk` |
| `0x657c8` | `AlsaPlayer::worker` |
| `0x47504` | `StreamServer::onChunkRead` |
| `0x2ab00` | `TimeReSync_thread` |

## 未知与不要推断的事

- 没有运行音箱，没有看到真实对时包或 ALSA 写入。
- `getPlayerChunk` 除以 1000 之后的单位，与硬校正立即数 1000/5000 相对日志 `> 1`/`> 5` 的关系，见上，未闭合。
- `suddenCount` 在本库里只被加减，没有读它来改变播放。
- `TimeReSync` 的信号量没有投递者。
- iotdcm 组回调如何设置 `is_Server` 在 `/usr/bin/mdplay`，本文没有反编译该可执行文件。日志字符串显示 `HandleMultiroomCreate` / `HandleStereoTrackChange` 决定角色和声道。
- `libiotdcm.so` 与 `libiotdcm_mdplay.so` 不是同一构建：后者带 XMD，前者带 `iotdcm_query_send_delay_time` 一类延迟统计。延迟统计不是样本对齐。
