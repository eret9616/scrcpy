# UDP Transport — Working Notes

实验性分支：把 scrcpy 的视频/音频流从 TCP（over adb）改成 UDP，解决无线传输丢包重传导致的卡顿。

> ⚠️ 实验代码，不打算合并回上游。

---

## 1. 为什么要做这个

scrcpy 当前的视频流路径：

```
手机 MediaCodec 硬编 H.264/H.265
   ↓ 写入 abstract LocalSocket "scrcpy_xxx"
adbd（adb 协议 over TCP）
   ↓ 通过 adb forward / reverse 转发
mac adb client → scrcpy 客户端 demuxer → 解码 → 显示
```

**痛点**：跨子网无线场景下，TCP 任何丢包都会触发重传 + HOL blocking，导致画面卡顿。详细分析见上游 issue [#1379](https://github.com/Genymobile/scrcpy/issues/1379)。

实测对比（同一台一加 PGZ110）：

| 路径 | 60fps + 15Mbps + 1920p | 30fps + 10Mbps + 1920p |
|---|---|---|
| USB | 丝滑 | 丝滑 |
| 无线 同子网 WiFi | 偶尔抖动 | 流畅 |
| 无线 跨子网企业网 | **明显卡顿** | 流畅但延迟可感 |

---

## 2. 现有代码关键位置

### Server 端（Android Java）

| 文件 | 作用 |
|---|---|
| `server/src/main/java/com/genymobile/scrcpy/Server.java` | 入口 main() |
| `server/src/main/java/com/genymobile/scrcpy/device/DesktopConnection.java` | **socket 抽象层**，目前用 `LocalServerSocket` 监听 abstract namespace |
| `server/src/main/java/com/genymobile/scrcpy/video/*` | 视频编码 + 写入 socket |

关键代码：`DesktopConnection.java:65-83` 是 `LocalServerSocket.accept()` 接受 video / audio / control 三个 stream。

### Client 端（C）

| 文件 | 作用 |
|---|---|
| `app/src/server.c` | server 启动 + adb tunnel + socket 建立 |
| `app/src/server.c:589 sc_server_connect_to()` | 拿到 3 个 socket（video/audio/control） |
| `app/src/server.c:869 sc_server_connect_to_tcpip()` | `--port` 模式，绕过 adb tunnel 直接 TCP |
| `app/src/demuxer.c` | 视频/音频流解封装：`[size header][packet data]` 循环 |
| `app/src/receiver.c` / `controller.c` | 控制通道 |
| `app/src/cli.c` | CLI 参数解析（加新参数从这里入手） |

### 流格式（demuxer.c 当前读取的）

```
Video stream:
[8 bytes header: pts + size + flags] [N bytes packet] (循环)

Audio stream:
[8 bytes header] [N bytes opus/aac frame] (循环)

Control stream:
双向，分别 controller.c (send) / receiver.c (recv)
```

---

## 3. 改造方案候选

### 方案 A：最小可行 — Raw UDP + 丢帧策略（推荐起步）

**思路**：
- Server 监听 `DatagramSocket` 端口 27184
- 每个视频 packet 分片成 N 个 UDP 数据报（MTU ~1400 字节）
- 加 header：`[stream_id | frame_seq | total_chunks | chunk_idx | flags]`
- Client 收齐一帧的所有 chunk 才递交给 demuxer；缺一个就**丢弃整帧**，等下一个关键帧
- 控制信号继续走 adb TCP（控制信号低带宽，不需要 UDP）

**工作量估算**：
- Server: 新增 `UdpDesktopConnection.java` 约 200 行
- Client: 新增 `udp_demuxer.c` 约 400 行
- CLI: 加 `--udp-host=IP --udp-port=N` 选项约 50 行

**优点**：简单，能跑起来快  
**缺点**：丢包率高时画面糊，依赖编码器频繁发关键帧

### 方案 B：KCP（中等工作量）

用 [skywind3000/kcp](https://github.com/skywind3000/kcp) 这个成熟的可靠 UDP 库（C 实现，约 1000 行）：

- Server 端用 [kcp-java](https://github.com/hkalyan/kcp-netty) 或 JNI 调用 C 版
- Client 端直接链接 kcp
- 自动重传 + 流控，但 ARQ 模型还是会有"丢包 → 重传 → 等"，对视频流不是最优

**优点**：代码量小，库已成熟  
**缺点**：本质还是 ARQ，跟 TCP 比改善有限

### 方案 C：自定义 FEC + NACK（理想方案）

参考 WebRTC 的做法：

- 视频帧 + 冗余 FEC 包（Reed-Solomon 或 XOR）
- 丢包优先用 FEC 恢复
- FEC 也救不回来就发 NACK 请求重发或请求关键帧
- 拥塞控制：用 GCC（Google Congestion Control）算法或简化版

**优点**：理论上等同 WebRTC 体验  
**缺点**：工作量大（2-3 周专心做），需要测试基础设施

### 方案 D：直接用 WebRTC

把 scrcpy-server 改成 WebRTC peer，client 用 libdatachannel 或 GStreamer webrtcbin。

**优点**：直接拿到工业级实现  
**缺点**：架构大改，依赖膨胀（libwebrtc / libdatachannel 都大），还需要信令通道（虽然 adb 可以充当）

---

## 4. 建议 Roadmap

按"最小可行 → 迭代优化"路线，先有东西能跑：

### M0：基线 + 测试工具
- [ ] 写一个网络劣化工具脚本（用 `pfctl` / `dummynet` 注入丢包和延迟）
- [ ] 录制 baseline：在 0%、1%、3%、5% 丢包率下 TCP 版的卡顿表现
- [ ] 把测试数据列成对比表（视觉评分 + 帧率统计）

### M1：方案 A 跑起来
- [ ] Server 加 `--udp-port=N` 参数，启动 UDP socket
- [ ] 协议：定义 packet header 格式
- [ ] Client 加 `--udp-host=IP --udp-port=N` 参数
- [ ] 实现 chunk 重组 + 丢帧策略
- [ ] 跑通端到端，至少能看到画面

### M2：调优
- [ ] 编码器 keyframe interval 调到 1-2 秒（默认更长会让丢帧后糊得久）
- [ ] 加 PLR（Packet Loss Rate）统计 + log
- [ ] 测试相同丢包率下跟 TCP 对比

### M3：加 NACK（可选）
- [ ] Client 每帧丢失 chunk 时发 NACK
- [ ] Server 维护小缓冲（最近 N 个 packet），收到 NACK 立即重发
- [ ] 测试效果

### M4：FEC（可选，难度高）
- [ ] 引入 XOR 或 Reed-Solomon FEC
- [ ] 每 K 个 packet 加 M 个冗余包
- [ ] 测试在高丢包下能否完全无感

---

## 5. 协议草案（方案 A 用）

UDP packet 头部固定 12 字节：

```
0       1       2       3
+-------+-------+-------+-------+
| ver   | type  | flags |  rsvd |  (4 bytes)
+-------+-------+-------+-------+
|         frame_seq (u32)       |  (4 bytes，每个 frame 一个序号)
+-------+-------+-------+-------+
|  chunk_idx (u16) | total (u16) |  (4 bytes)
+-------+-------+-------+-------+
|         payload (≤ MTU-12)     |
+-------------------------------+
```

字段：
- `type`: 0=video, 1=audio, 2=keepalive
- `flags`: bit0=keyframe (整帧关键帧标记，所有 chunk 都打这个标)
- `frame_seq`: 单调递增的帧编号
- `chunk_idx` / `total`: 这是该帧的第几片 / 共多少片

Client 维护一个滑动窗口缓冲：

```c
struct frame_buffer {
    uint32_t frame_seq;
    uint16_t total_chunks;
    uint16_t received_chunks;
    uint8_t  is_keyframe;
    uint8_t  chunks[MAX_CHUNKS][CHUNK_SIZE];
    bool     chunk_received[MAX_CHUNKS];
};
```

策略：
- 收到所有 chunks → 重组帧 → 喂给 demuxer
- 等 100ms 还差几个 chunk → 丢弃整帧（如果是关键帧，可选发请求重传）
- 维护最近 16 帧的缓冲，丢晚到的 chunk

---

## 6. 测试方法

### 网络劣化注入

```bash
# macOS 用 pfctl + dummynet
sudo dnctl pipe 1 config plr 0.03 delay 5ms bw 10Mbit/s
# 应用规则
sudo pfctl -e
echo "dummynet in proto udp from any to any port 27184 pipe 1" | sudo pfctl -f -
```

### 性能指标

| 指标 | 测量方法 |
|---|---|
| 端到端延迟 | 手机上闪屏，看到画面的时间差（需相机辅助） |
| 帧率稳定性 | client 端记录每帧的 pts 间隔，统计 stddev |
| 卡顿次数 | client 端记录 >50ms 的帧间隔 |
| 画面质量 | PSNR 对比 / 主观评分 |

---

## 7. 已知风险和坑

1. **Android 权限**：scrcpy-server 是 `app_process` 跑的，理论上有 INTERNET 权限。但某些 ROM 可能限制 UDP 高端口监听 — 实测确认。
2. **NAT/防火墙**：UDP 在企业网可能被封某些端口。建议默认用 27184-27200 范围。
3. **手机休眠**：UDP 没有连接概念，手机锁屏后 socket 可能被系统杀。需要 keepalive 包。
4. **MTU 探测**：默认按 1400 切，但部分网络可能更小。第一版可以写死 1200 安全值。
5. **mac firewall**：macOS 防火墙默认阻 incoming UDP，需要测试是否要开规则。

---

## 8. 参考

- [scrcpy issue #1379 UDP/RTP support](https://github.com/Genymobile/scrcpy/issues/1379)
- [KCP 协议](https://github.com/skywind3000/kcp)
- [WebRTC FEC RFC 5109](https://datatracker.ietf.org/doc/html/rfc5109)
- [Google Congestion Control draft](https://datatracker.ietf.org/doc/html/draft-ietf-rmcat-gcc)
- [libdatachannel - 轻量 WebRTC C++ 库](https://github.com/paullouisageneau/libdatachannel)

---

## 9. 开始动手

```bash
# 启动开发环境
cd ~/code/scrcpy
git checkout feat/udp-transport

# 上游同步（之后）
git fetch upstream
git rebase upstream/master
```

构建：见 [doc/build.md](doc/build.md)。

第一步建议：先在 `app/src/cli.c` 加 `--udp-port` 参数解析（最简单的入口，不动核心逻辑），跑通编译。然后再去碰 socket 那一层。
