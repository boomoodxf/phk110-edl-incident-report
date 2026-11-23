# 事故详细报告：从刷入 TWRP 到 19:33 黑屏

**设备**：一加 Ace 2（PHK110 / OP5913L1 / SM8475 / 12 GB / 256 GB / ColorOS 16.0.5.1301）
**时间窗**：2026-11-22 约 17:22 — 19:33（北京时间）
**结果**：设备在 `set_active b` 后未启动，直接落入 Qualcomm 9008（EDL），按任何键无响应，屏幕黑屏。

---

## 0. 一句话结论

P0（缩容 + 建 darwin + 灌数据卷）**全部成功且逐字节验证通过**；
失败发生在最后一步——**把 HakiOS 的 `boot_b` 镜像作为活动槽冷启动**，设备没有回退到 a 槽，而是进入 EDL。
本地所有 EDL 恢复通道（OplusEdlTool 本地/云端引导、fh_loader、QSaharaServer、自写 Sahara 客户端）均无法推进，设备停在异常 Sahara 会话。

---

## 1. 时间线总表

| 时刻 | 动作 | 结果 |
|---|---|---|
| ~17:22 | 上一轮 `fastboot boot TWRP` 失败导致重启循环，执行 `set_active a` 恢复 | ✅ Android 正常启动 |
| ~17:24 | 机主自行刷入 TWRP 并进入恢复模式 | ✅ `adb` 显示 `recovery`，TWRP 3.7.1_12-Udon-Color597-V2.1 |
| 17:25 | 检查 TWRP 能力与 /data 可见性 | ✅ root 全能力（`CapEff=000001ffffffffff`），/data 以 f2fs 挂在 `dm-17` |
| 17:25 | `f2fs-resize /data probe` | ✅ 返回 `EINVAL`（表示 ioctl 存在） |
| 17:26:43 | 采集缩容前 GPT / f2fs 超级块 / segment_info | ✅ 4 个文件落盘 |
| 17:26:5x | `userdata-carve-phk110.py --dry-run` | ✅ 预检全过（last data segment 8083 ≪ 102224） |
| 17:27:0x | **执行在线缩容**：`/tmp/fr /data 52514816` | ✅ `done`；`/data` 220G → **200G**，13G 数据保留 |
| 17:27:01 | 读取缩容后超级块 | ✅ `block_count=52,515,087`、`segs_main=102,224`、`main_blkaddr=176,128`（与作者预期逐项相等） |
| 17:27:07 | 生成新 GPT（含 darwin） | ✅ `gpt-head.new 24268F31…`、`gpt-tail.new A0515AFE…` |
| 17:27:1x | 写入 GPT（主表 LBA2→LBA1；备表 61514747 起 5 块） | ✅ dd 全部 OKAY |
| 17:27:21 | 读回两份 GPT | ✅ SHA256 **与生成文件逐字节一致** |
| ~17:28 | 重启回 Android | ✅ `boot_completed=1`，`PHK110_16.0.5.1301`，槽位 `_a` |
| ~17:29 | 校验分区与数据 | ✅ `/data` 200G；`darwin -> sda15`，**21,474,816,000 B = 20.000 GiB**；`/sdcard` 数据完好 |
| 17:34 | `FLASHBOOT=1` 重新构建镜像完成 | ✅ `flashboot.apfs` 977 MB、`system.apfs` 2.4 GB、`root.apfs` 4 MB |
| ~17:5x–18:10 | `mkdatavol.py` → `simg2img` → 截取 | ✅ `darwin.simg` 53,464 B；raw 20 GiB；数据卷 **13,109,276,672 B** |
| 18:12–18:15 | 数据卷镜像复制到 D 盘 | ✅ 13,109,276,672 B |
| 18:17–18:31 | 经 `adb shell` stdin 流式写入 darwin 块 2,042,368 起 | ✅ 13,109,276,672 B，840.7 s，15 MB/s |
| 18:36 | 检查卷头签名 | ✅ 块 2,042,368 偏移 1024 = **`HX`**（HFS+ 日志式） |
| 18:37–19:11 | 整卷读回 13 GB 计算 SHA256 | ✅ **MATCH**（源文件 = 读回） |
| 19:27 | 校验并刷入 `boot_b`（`udon-hakios-grafted-v101-daily-gpu.img`，SHA `68c1c6af…`，192 MiB） | ✅ `Writing 'boot_b' OKAY` |
| 19:27 | `fastboot set_active b` | ✅ `Setting current slot to 'b' OKAY` |
| 19:28:31 | 重启后第一次探测 | ❌ `adb`/`fastboot` 均无设备；USB 出现 **Qualcomm HS-USB QDLoader 9008 (COM3)** |
| 19:28–19:31 | 反复探测（6 次，每 20 s） | ❌ 始终 9008，无 fastboot |
| ~19:32 | 机主长按电源 20–30 s / 音量组合 | ❌ 无反应 |
| 19:33 | 机主回报「黑屏没反应」 | ❌ 停留在 EDL |

---

## 2. 阶段一：TWRP 中的分区手术（17:24–17:27）

### 2.1 环境与关键发现

```
adb devices → 8834a83f  recovery  product:OP5913L1 model:PHK110 device:OP5913L1
getprop ro.twrp.version → 3.7.1_12-Udon-Color597-V2.1
grep " /data " /proc/mounts → /dev/block/dm-17 /data f2fs rw,...
id → uid=0(root) context=u:r:su:s0
CapEff: 000001ffffffffff   ← 完整能力集（Android 里 Magisk 的 su 只有 0x80）
```

这一步是关键：**Android 里 `su` 的能力边界只有 `CAP_SETPCAP`，而 TWRP 里是完整的**，所以 `F2FS_IOC_RESIZE_FS` 在 TWRP 里可用。

同时确认了数据是“元数据加密”的：

- 裸分区 `userdata` 偏移 1024 的超级块是密文（`4b 88 fe a9 …`）；
- 真正的 f2fs 超级块要通过 `dm-17` 读，`magic=0xf2f52010`。

### 2.2 缩容（不可逆步骤 1）

```
adb shell '/tmp/fr /data 52514816'
→ f2fs-resize: F2FS_IOC_RESIZE_FS to 52514816 blocks: done
df -h /data:  220G → 200G（已用 13G 不变）
```

缩容后超级块核对：

| 字段 | 值 | 作者预期 | |
|---|---|---|---|
| block_count | 52,515,087 | 52,515,087 | ✅ |
| segs_main | 102,224 | 102,224 | ✅ |
| main_blkaddr | 176,128 | 176,128 | ✅ |

### 2.3 GPT 重写（不可逆步骤 2）

工具计算（`userdata-carve-phk110.py`，作者原版，未改）：

```
userdata: lba 3756780..61514746 (220.33 GiB) -> 3756780..56271866 (200.33 GiB，正好是 f2fs)
darwin:   lba 56271872..61514746 (20.000 GiB, 5242875 blocks of 4096), entry 15
         type APFS container, unique GUID 6e9d0187-62e7-4887-b50e-fb241bf4be6d
新表只改 entry 14、15；主备两份 CRC 全对、内容一致
```

写入（严格按作者要求的“先表后头”）：

```bash
dd if=/tmp/gpt-head.new of=/dev/block/sda bs=4096 skip=2 seek=2 count=1 conv=notrunc,fsync
dd if=/tmp/gpt-head.new of=/dev/block/sda bs=4096 skip=1 seek=1 count=1 conv=notrunc,fsync
dd if=/tmp/gpt-tail.new of=/dev/block/sda bs=4096 skip=3 seek=61514747 count=5 conv=notrunc,fsync
```

读回校验：

| 文件 | SHA256 | 读回 SHA256 |
|---|---|---|
| gpt-head.new | `24268F31EE10F6CCB9E2124CC61B8ACD0BEB35CA4618D115051B77D29ACFEFD0` | 同左 ✅ |
| gpt-tail.new | `A0515AFECC98338798A1ACA969214B6561870B4112C8C03D37FB36F400936F0A` | 同左 ✅ |

### 2.4 回 Android 验证

```
boot_completed=1
PHK110_16.0.5.1301(CN01)   槽位 _a
df /data → 200G（已用 13G）
/dev/block/by-name/darwin -> /dev/block/sda15
blockdev --getsize64 darwin → 21,474,816,000
/sdcard 目录、Download 内文件、DCIM 均在
```

**P0 到这里成功。** 此时手机完全可用，只是多了 20 GiB 空分区。

---

## 3. 阶段二：生成并写入 HFS+ 数据卷（17:5x–19:11）

### 3.1 生成

```
mkdatavol.py → darwin.simg（53,464 B 稀疏镜像，内含 12 GB HFS+ 卷，起点 darwin 块 2,042,368，长度 3,200,507 块）
simg2img      → 21,474,816,000 B（整个 darwin）
tail 截取     → 13,109,276,672 B（= 3,200,507 × 4096，正好到分区末块）
SHA256        → 88525133a899630a1284fb8113b1628308e6b9d0b40b291e508510dc122d81f4
```

### 3.2 写入方式的一个细节

Android 里 root **读不了 `/sdcard`**（SELinux：

```
su -c 'head -c 8 /sdcard/Download/f2fs-resize' → Permission denied
）,
```

也**写不了 `/data/local/tmp`**（`chmod: Operation not permitted`）。
所以没有用作者的 `darwin-write-phk110.py`（它要先把镜像 push 到 `/data/local/tmp`），改成：

```python
subprocess.Popen([adb, "shell", "su -c 'dd of=/dev/block/by-name/darwin bs=4096 seek=2042368 conv=notrunc,fsync'"], stdin=open(镜像,"rb"))
```

即把镜像**从 PC 直接流进 root 的 dd**，绕开两个不可写目录。

结果：

```
3200299+228 records
13109276672 bytes (12 G) copied, 840.706 s, 15 M/s
```

### 3.3 校验

卷头（darwin 块 2,042,368，偏移 1024）：

```
HX
48 58 00 05 80 00 21 00 31 30 2e 30 00 00 00 63   ← 'HX'+版本5+属性+「10.0」
```

整卷读回（13,109,276,672 B）：

```
got  88525133a899630a1284fb8113b1628308e6b9d0b40b291e508510dc122d81f4
want 88525133a899630a1284fb8113b1628308e6b9d0b40b291e508510dc122d81f4
MATCH
```

**这一步也成功了。** 此时 darwin 里有一个可用的 HFS+ 空卷，Android 仍可正常启动（写卷不需要重启）。

---

## 4. 阶段三：启动测试与 EDL（19:27–19:33）

### 4.1 动作

```
镜像校验：udon-hakios-grafted-v101-daily-gpu.img
          201,326,592 B，SHA256 68c1c6af5561dc992400f53468db6ee09308445311f842549779cb76eb95e3cb  ✅ 与发布页一致
fastboot flash boot_b …   → Sending OKAY / Writing 'boot_b' OKAY
fastboot set_active b     → OKAY
fastboot reboot           → OKAY
```

### 4.2 结果

```
19:28:31  adb=[]  usb=[Qualcomm HS-USB QDLoader 9008 (COM3)]
19:28:53  同上
19:29:15  同上
19:29:37  同上
19:30:00  同上
19:30:22  同上
```

`USB\VID_05C6&PID_9008&…`，`fastboot devices` 为空。

### 4.3 关键差异：不是“兜底回安卓”，而是直接进 EDL

作者文档（v0.4 release notes）写的是：未做 P0 时 init 链挂载失败会“兜底回安卓（安全失败）”。
实际发生的是：**设备在 b 槽启动早期就落到 9008，没有走 ABL 的 A/B 重试回退**。

可能原因（按可能性排序）：

1. **镜像期望“闪存启动”布局，而 darwin 前 5.63 GB 是空的。**
   作者的 `darwin-write-phk110.py` 里 `PROTECTED = 1,374,208` 块（5,628,755,968 B）注明是 `rootfs.apfs`（Darwin 系统卷）的区域，并说“从未被写”。
   也就是说：**darwin 的前 5.63 GB 应该放着 Darwin 根卷，而我们只写了后半段的数据卷**（块 2,042,368 起）。
   如果这版镜像是 flash boot（根在 darwin），内核会在挂载根卷时失败——早期 panic 有可能直接触发 bootloader 的 EDL 路径。
2. **内存图不匹配**：镜像按 16 GB 机型调的内存布局（`PAYLOAD_BASE=0xF4000000`、`FD_BASE=0xA7100000`），本机是 12 GB。
3. **graft 头部/vbmeta 与 b 槽环境不匹配**：镜像头取自作者自己的原厂 boot，不是本机的（本机 `boot_b` 之前被我们保留为出厂状态）。
4. **该镜像本身只适合作者的 b 槽实验环境**，冷启动路径没有做过异机验证。

### 4.4 EDL 后的救援尝试（19:33 之后，超出本报告窗口，简要记录）

| 手段 | 结果 |
|---|---|
| 长按电源 60 s / 电源+音量下 / 电源+音量上 | 无效 |
| 按住音量上下插 USB | 设备重新枚举，但 Sahara 不再发 HELLO |
| OplusEdlTool + 本地 `OPPO_SM8475_V3.0.melf`+Digest+Sign | 「引导发送失败」 |
| OplusEdlTool 云端引导 | 同样失败 |
| OplusEdlTool + PHK110 专用 `prog_firehose_ddr.elf` | 同样失败 |
| `fh_loader`（QPST） | 报“目标在 SAHARA 模式，需先送 DeviceProgrammer” |
| `QSaharaServer -u 3 -s 13:prog_firehose_ddr.elf` | 端口能开，读不到 HELLO（0 字节） |
| 自写 Sahara 客户端（HELLO/DONE/RESET） | 一律只回 `04000000 10000000 0d000000 01000000`（END_IMAGE_TX），状态机不推进 |

---

## 5. 数据与资产现状

**手机闪存内（未损坏，待救砖后可继续）**

- `userdata`：200 GiB f2fs，原有数据完好（缩容前已验证尾部 20 GiB 无有效数据）；
- `darwin`（sda15）：20 GiB，其中块 2,042,368 起是已校验的 13 GB HFS+ 数据卷；
- `boot_a`：Magisk 31.1 补丁版（原厂镜像有备份）；
- 活动槽：**b**（故障点，需切回 a）。

**PC 侧备份**

| 文件 | 说明 | SHA256 |
|---|---|---|
| `work/fw/boot-1301.img` | 原厂 boot | `205680da…47f4b7` |
| `work/gpt-primary.bin` / `gpt-backup.bin` | P0 前 GPT | `49E0EBF1…` / `ead672e5…` |
| `work/p0twrp/gpt-head.new` / `gpt-tail.new` | P0 后 GPT | `24268F31…` / `A0515AFE…` |
| `work/img/udon-hakios-grafted-v101-daily-gpu.img` | 触发故障的镜像 | `68c1c6af…` |

**WSL（D:\wsl\Ubuntu）**：完整 phk110-port 源码、解包后的 IPSW 用户态、构建产物
`system.apfs`(2.4 GB) / `root.apfs`(4 MB) / `flashboot.apfs`(977 MB)、`darwin-vol.raw`(13.1 GB)。

---

## 6. 复盘：本可以做得更好的地方

1. **启动测试前应先补齐 darwin 的“系统卷”区域。**
   作者工具把前 1,374,208 块标为保护区，说明 flash 启动需要那里的 rootfs；我们跳过了它，只写了数据卷，这是最可能的设计性缺口。
2. **`fastboot boot` 这条“只试不写”的安全通道不存在。**
   一加 ABL 拒绝内存引导（`Too many links`），导致所有启动验证都只能靠写槽位。
3. **EDL 兜底没有事先验证。**
   我们事先知道本地有 firehose 工具，但没有验证“这台机器 + 这个固件版本”能否走通授权；等真出事时才发现 OplusEdlTool 的引导发送过不去。
4. **文档里的“安全失败”承诺不可跨机型外推。**
   作者的“兜底回安卓”是在他自己的环境（16 GB、已做完整 P0 含 rootfs、相同固件）成立的，不宜当成通用保证。
5. 事后看，更稳的顺序应是：**先在 WSL/QEMU 里验证这版镜像能起 → 再补齐 darwin 系统卷 → 最后才切活动槽**；或者干脆只做 P0、不碰 `boot_b`。

---

## 7. 下一步

1. **恢复手机**（优先）：授权 EDL/OFP 线刷，把活动槽切回 a、修复引导；明确要求保留 `/data` 与 `darwin`。
2. 恢复后第一时间：`fastboot set_active a` → 确认 Android 起来 → 再考虑是否继续项目。
3. 若继续：先补 darwin 前 5.63 GB 的 Darwin 根卷（源码与构建环境都还在），并重新评估 12 GB 内存的适配问题。
