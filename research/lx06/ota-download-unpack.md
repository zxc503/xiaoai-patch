# LX06 OTA 下载与 rootfs 解包

供后续 agent / 人工按本文复现：从小米 CDN 下载 LX06 固件，解出明文 `root.squashfs`，并展开到工作区。

本文以仓库兼容目标版本 **LX06 1.74.10**（MD5 `fe6edc680d58620019c9afe4fd79c712`，类型 `firmware`）为例。

## 前置

- 工具：`curl`、`python3`、`unsquashfs`（`squashfs-tools`）
- 仓库内脚本：`tools/mico_firmware.py`
- 版本索引：`research/updates.md`

## 1. 拼下载 URL

规则（见 `research/updates.md`）：

```text
https://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/{MODEL}/mico_{TYPE}_{MD5_SUFFIX}_{VERSION}.bin
```

- `{MODEL}`：如 `lx06`
- `{TYPE}`：表中的 Update type（本例为 `firmware`；也有 `all`）
- `{MD5_SUFFIX}`：完整 MD5 的**后 5 位**；2024 起部分机型/版本用**后 9 位**（LX06 1.74.10 用后 5 位即可）
- 备用 host：`http://bigota.miwifi.com`、`https://cdn.awsde0-fusion.fds.api.mi-img.com`

本例完整 MD5：`fe6edc680d58620019c9afe4fd79c712` → 后缀 `9c712`。

已知可用地址：

```text
https://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/lx06/mico_firmware_9c712_1.74.10.bin
```

探测示例（先 HEAD，200 再用）：

```bash
MD5=fe6edc680d58620019c9afe4fd79c712
SHORT5=${MD5: -5}
SHORT9=${MD5: -9}
MODEL=lx06
VER=1.74.10
TYPE=firmware
for H in \
  https://cdn.cnbj1.fds.api.mi-img.com \
  http://bigota.miwifi.com \
  https://cdn.awsde0-fusion.fds.api.mi-img.com
do
  for S in "$SHORT5" "$SHORT9"; do
    URL="$H/xiaoqiang/rom/$MODEL/mico_${TYPE}_${S}_${VER}.bin"
    CODE=$(curl -sI -L --max-time 15 -o /dev/null -w "%{http_code}" "$URL")
    echo "$CODE $URL"
  done
done
```

## 2. 下载并校验

```bash
mkdir -p /tmp/lx06-fw && cd /tmp/lx06-fw
URL="https://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/lx06/mico_firmware_9c712_1.74.10.bin"
# 文件名必须保留 MD5 后缀段，供 mico_firmware.py 校验（勿改名为 fw.bin）
curl -L --fail -o mico_firmware_9c712_1.74.10.bin "$URL"
md5sum mico_firmware_9c712_1.74.10.bin
# 期望: fe6edc680d58620019c9afe4fd79c712
```

## 3. 查看 / 解包 OTA（HDR1）

```bash
# 仅校验与展示
python3 /workspace/tools/mico_firmware.py -s mico_firmware_9c712_1.74.10.bin

# 解到空目录（目标目录必须尚不存在，否则脚本 os.mkdir 会失败）
rm -rf /tmp/lx06-fw/out
python3 /workspace/tools/mico_firmware.py \
  -e mico_firmware_9c712_1.74.10.bin \
  -d /tmp/lx06-fw/out
```

期望：

| 项 | 值 |
|----|-----|
| magic | `0x31524448`（HDR1，明文；`0x32524448` HDR2 为加密包） |
| 段 | `mico_version`、`root.squashfs`、`boot.img` |

若脚本因目录已存在失败，可手工按段头拆包（段 magic `0xbabe`，名称在偏移 +16）：

```bash
python3 - <<'PY'
import struct, os
path = "/tmp/lx06-fw/mico_firmware_9c712_1.74.10.bin"
data = open(path, "rb").read()
outdir = "/tmp/lx06-fw/segments"
os.makedirs(outdir, exist_ok=True)
babe = struct.pack("<I", 0xbabe)
pos = 0
while True:
    i = data.find(babe, pos)
    if i < 0:
        break
    if i + 48 <= len(data):
        name = data[i + 16 : i + 48]
        if name[:1].isalpha() and all(32 <= b < 127 or b == 0 for b in name):
            n = name.split(b"\0", 1)[0].decode()
            length = struct.unpack_from("<I", data, i + 8)[0]
            blob = data[i + 48 : i + 48 + length]
            open(f"{outdir}/{n}", "wb").write(blob)
            print(n, length, blob[:4])
    pos = i + 1
PY
```

确认 squashfs：

```bash
file /tmp/lx06-fw/segments/root.squashfs
# 或 out/root.squashfs
# 应含: Squashfs filesystem ... little endian
unsquashfs -s /tmp/lx06-fw/segments/root.squashfs
```

## 4. 展开 rootfs 到工作区

```bash
ROOT_SQ=/tmp/lx06-fw/segments/root.squashfs
# 若用 -d out 解包成功，则改为 /tmp/lx06-fw/out/root.squashfs

mkdir -p /workspace/lx06-fw
unsquashfs -d /workspace/lx06-fw/rootfs -f "$ROOT_SQ"
cp -a "$ROOT_SQ" /workspace/lx06-fw/root.squashfs
cp -a /tmp/lx06-fw/segments/mico_version /workspace/lx06-fw/mico_version \
  2>/dev/null || cp -a /tmp/lx06-fw/out/mico_version /workspace/lx06-fw/mico_version
```

结果布局：

```text
/workspace/lx06-fw/
  mico_version          # ROM 1.74.10 / HARDWARE LX06
  root.squashfs         # ~36MB 镜像
  rootfs/               # unsquashfs 展开树
```

## 5. 冒烟检查（多音响相关闭源库）

原厂删除名单见 `scripts/24_remove_property_bin.sh`。明文 rootfs 中应存在：

```bash
ls -lh \
  /workspace/lx06-fw/rootfs/usr/lib/libmdplay.so \
  /workspace/lx06-fw/rootfs/usr/lib/libiotdcm.so \
  /workspace/lx06-fw/rootfs/usr/lib/libiotdcm_mdplay.so
```

`mico_version` 中应有：

```text
option ROM '1.74.10'
option HARDWARE 'LX06'
```

## 注意

1. **不要**把 `/workspace/lx06-fw/rootfs` 或 `*.bin` / `*.squashfs` 提交进 git；仓库 `.gitignore` 已忽略 `/lx06`、`/firmware`、`/mico_*.bin` 等。`lx06-fw/` 若需本地保留，勿 `git add` 大体积二进制。
2. **HDR2**（如部分 L16A `all` 包 magic `0x32524448`）虽可能含名为 `rootfs.squashfs` 的段，但内容加密，`unsquashfs` 会失败；LX06 1.74.10 `firmware` 为 HDR1，可直接解。
3. `mico_firmware.py -d` 的目标目录必须不存在；文件名需含 MD5 后缀，否则校验会报错（可用 `-i` 忽略 hash，但不推荐）。
4. 其它版本：查 `research/updates.md` 换 MD5 / TYPE / VERSION，重复第 1–4 步；优先选 magic 为 HDR1 的包。
