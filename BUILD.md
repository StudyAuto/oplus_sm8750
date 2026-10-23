# 一加 13（SM8750）OKI 内核构建与复用指南

> 本文记录一次完整的实战流程：从识别设备、创建公开仓库、修复上游阻断，
> 到产出可用 AnyKernel3 刷机包并在真机验证。下次复用时按 §2 直接照做即可。

---

## 1. 目标设备与基线

| 项 | 值 |
|---|---|
| 机型 | 一加 13（`PJZ110` / `OP5D0DL1`） |
| SoC | SM8750 `sun` |
| 系统 | `PJZ110_16.0.10.501(CN01)`，Android 16 |
| 安全补丁 | `2026-08-01` |
| 当前槽位 | `_a` |
| 内核基线 | 6.6.118，KMI `android15-8` |
| CPU ABI | `arm64-v8a` |

### 1.1 原厂内核版本串（重要基准）

从官方 `boot.img` 反解，**不是**读设备属性：

```
Linux version 6.6.118-android15-8-g93e223c276e7-abogki500782043-4k
  (kleaf@build-host)
  (Android (11368308, +pgo, +bolt, +lto, +mlgo, based on r510928)
   clang version 18.0.0
   (https://android.googlesource.com/toolchain/llvm-project 477610d4d0d988e69dbc3fae4fe86bff3f07f2b5),
   LLD 18.0.0)
  #1 SMP PREEMPT Wed Apr  8 15:08:30 UTC 2026
```

拆解成三个可变字段（其余段无法用构建变量改写）：

| 字段 | 值 | 来源变量 |
|---|---|---|
| release 尾部 | `android15-8-g93e223c276e7-abogki500782043-4k` | `KERNEL_NAME` → `CONFIG_LOCALVERSION` |
| 编译者 | `kleaf@build-host` | `KBUILD_BUILD_USER` / `KBUILD_BUILD_HOST` |
| 时间 | `Wed Apr  8 15:08:30 UTC 2026` | `KBUILD_BUILD_TIMESTAMP` |

编译器段（`Android (11368308, ...)` + git URL）由 toolchain 决定，无法伪装；
但本项目用的 `clang-r510928` 与原厂同源，故天然一致。

### 1.2 原厂关键配置

```
CONFIG_LOCALVERSION="-4k"          CONFIG_LOCALVERSION_AUTO=y
CONFIG_KALLSYMS=y                  CONFIG_KALLSYMS_ALL=y
CONFIG_HEADERS_INSTALL=y           CONFIG_TRIM_UNUSED_KSYMS=y
CONFIG_MODULE_SCMVERSION=y
CONFIG_ZRAM=m                      CONFIG_ZRAM_DEF_COMP="lzo-rle"
CONFIG_MQ_IOSCHED_DEADLINE=y       CONFIG_MQ_IOSCHED_KYBER=y
```

原厂**没有** `CONFIG_KSU` / `CONFIG_KPM` / `CONFIG_MQ_IOSCHED_ADIOS`。

### 1.3 官方启动链备份

Release `stock-backup-16.0.10.501`（私库）内含：

```
boot.img.gz          96 MB    回滚主力
init_boot.img.gz      8 MB    含 recovery
vendor_boot.img.gz   96 MB    ramdisk + dtb
dtbo.img.gz          24 MB
vbmeta*.img.gz       12 KB / 4 KB
```

来源：官方 OTA `payload.bin`，SHA256 `91c1fa14c96d4765815e9434dbc4a6a6`。

---

## 2. 快速复用（三步）

### 2.1 确认设备信息

```sh
ADB=/mnt/c/Windows/System32/adb.exe      # WSL 下直接用 Windows 的 adb
$ADB devices -l
$ADB shell getprop ro.product.device       # OP5D0DL1
$ADB shell getprop ro.build.display.id     # PJZ110_16.0.10.501(CN01)
$ADB shell cat /proc/version               # 当前内核串
$ADB shell getprop ro.boot.slot_suffix     # _a
```

### 2.2 触发构建

```sh
cd /home/pigmomo/pigmomo2026/oplus_sm8750
TOKEN=$(printf "protocol=https\nhost=github.com\n\n" \
  | git credential fill 2>/dev/null | sed -n 's/^password=//p')

curl -s -X POST \
  -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/StudyAuto/oplus_sm8750/actions/workflows/fastbuild_6.6.118.yml/dispatches \
  -d '{"ref":"main","inputs":{
    "ksu_type":"sukisu",
    "susfs_enable":"true",
    "kpm_enable":"true",
    "lz4_enable":"true",
    "lz4kd_enable":"false",
    "bbr_enable":"false",
    "droidspaces_enable":"false",
    "better_net":"false",
    "adios_enable":"true",
    "rekernel_enable":"false",
    "baseband_guard":"false",
    "ccache_update":"false",
    "ccache_debug":"false"
  }}' -w "HTTP:%{http_code}\n"
```

`HTTP:204` 即触发成功。查询进度：

```sh
curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/StudyAuto/oplus_sm8750/actions/runs?per_page=1" \
  | python3 -c "import json,sys;r=json.load(sys.stdin)['workflow_runs'][0];print(r['id'],r['status'],r['conclusion'])"
```

耗时参考：全新约 25–35 分钟；命中 ccache 约 12–18 分钟。

### 2.3 参数推荐值

| 输入 | 推荐 | 说明 |
|---|---|---|
| `ksu_type` | **`sukisu`** | 必选。SukiSU Ultra builtin，唯一带 KPM 管理界面的分支 |
| `susfs_enable` | `true` | 隐藏能力，需配合 `susfs4ksu` 模块下发配置 |
| `kpm_enable` | `true` | 用 SukiSU 官方 kpimg 注入真实 KPM，见 §4.7 |
| `lz4_enable` | `true` | lz4 1.10.0 + zstd 1.5.7（补丁已修正，见 §4.2） |
| `lz4kd_enable` | `false` | 与 lz4 同时开无额外收益 |
| `bbr_enable` | `false` | 手机端增加耗电 |
| `droidspaces_enable` | `false` | 容器支持，按需 |
| `better_net` | `false` | ipset/iptables 扩展，按需 |
| `adios_enable` | `true` | ADIOS IO 调度器 |
| `rekernel_enable` | **`false`** | ⚠️ 本源码树无 `CONFIG_REKERNEL` 定义，开了会被静默忽略 |
| `baseband_guard` | `false` | 基带保护，按需；`true` 会改 LSM 链 |
| `kernel_suffix` | 留空 | 留空才使用 `KERNEL_NAME` 的原厂后缀 |

---

## 3. 产物验证（必做）

**不要只看「构建成功」**。刷机前用以下步骤反解镜像确认功能真的编进去了。

### 3.1 拉取并解包

```sh
cd /tmp/opencode && rm -rf ak3 && mkdir ak3 && cd ak3
curl -sL -o a.zip "<release asset 直链>"
unzip -o -q a.zip && ls
```

### 3.2 反解内嵌配置

```sh
python3 - <<'PY'
import zlib, re
d = open('Image','rb').read()
for m in re.finditer(b'\x1f\x8b\x08', d):
    try:
        o = zlib.decompressobj(16 + zlib.MAX_WBITS)
        out = o.decompress(d[m.start():m.start()+3000000])
        if b'CONFIG_' in out and len(out) > 5000:
            open('cfg','wb').write(out); print('extracted', len(out)); break
    except Exception:
        pass
PY
grep -E "^CONFIG_(KPM|KSU|KSU_SUSFS|HEADERS_INSTALL|MQ_IOSCHED_ADIOS|LOCALVERSION)" cfg | sort
```

期望：

```
CONFIG_KPM=y
CONFIG_KSU=y
CONFIG_KSU_SUSFS=y   (+ 9 项子功能)
CONFIG_HEADERS_INSTALL=y
CONFIG_MQ_IOSCHED_ADIOS=y
CONFIG_LOCALVERSION="-android15-8-g93e223c276e7-abogki500782043-4k"
```

### 3.3 校验版本串与原厂一致

```sh
strings -a Image | grep -m1 "Linux version 6.6.118"
# 与原厂 boot.img 的 banner 逐字节比对（见 §1.1）
```

### 3.4 校验 KPM 注入

kpimg 注入后，`Image` 里搜不到字面量 `kpimg`，要按片段比对：

```sh
# kpimg 从 SukiSU-Ultra/SukiSU_KernelPatch_patch 下载
python3 - <<'PY'
img = open('Image','rb').read()
kp  = open('kpimg','rb').read()
assert b'sukisu_kpm_version' in kp, 'kpimg 来源错误'
PROBE = 65536                      # 该区域是代码段，kptools 不改写
pos = img.find(kp[PROBE:PROBE+256])
base = pos - PROBE
diff = sum(1 for a,b in zip(img[base:base+len(kp)], kp) if a != b)
print('base', hex(base), 'diff %d/%d = %.3f%%' % (diff, len(kp), 100*diff/len(kp)))
PY
```

期望：`diff` 约 110/294144 ≈ **0.038%**（差异全在 kpimg 头部前 500 字节，
是 kptools 写入的 superkey 与符号偏移）。

### 3.5 功能特征串

```sh
strings -a Image | grep -m1 "1.10.0"                       # lz4
strings -a Image | grep -m1 "1.5.7"                        # zstd
strings -a Image | grep -m1 "block/adios.c"                # ADIOS
strings -a Image | grep -m1 "sukisu_kpm_version"           # KPM 桩符号
```

---

## 4. 已修复的问题（踩坑记录）

按发现顺序排列，每条含根因与验证方式。

### 4.1 BakaSU 上游阻断 SUSFS 构建

**现象**

```
susfs_compat.mk:20: *** Please wait susfs update.  Stop.
```

**根因**：BakaSU 在 `48fa4bb7`（2026-10-10, "kernel: suspend susfs build"）
给 `kernel/tools/susfs_compat.mk` 末尾加了**无条件** `$(error)`。该文件由
`kernel/Kbuild` 的 `CONFIG_KSU_SUSFS` 分支无条件 include，所以只要开 SUSFS 必失败。

**处理**：`bakasu` 分支固定到 `BAKASU_REF=v4.2.0-rc3`——最后一个未加拦截的版本，
且其 `inline_hook_check.mk` 要求的 7 个 `ksu_handle_*` 钩子与 `susfs4oki`
补丁逐一对应。

> 注：最终路线改用 SukiSU builtin（§4.6），`bakasu` 仅作备选保留。

### 4.2 lz4 补丁 100% 静默失效

**现象**：日志有 `patch does not apply`，但构建继续，产物里没有 lz4 1.10.0。

**根因两层**：

1. `git apply` 是**全有或全无**：`lib/lz4/lz4hc_compress.c` 的删除段少了一行
   `EXPORT_SYMBOL(LZ4_resetStreamHC);`（本源码树比补丁基线多这一行），
   导致整个补丁包回滚。
2. 命令后接了 `|| true`，失败被吞掉。

**处理**：

- 重制 `zram_patch/001-lz4.patch`：删除段补上该 `EXPORT_SYMBOL`，行数 768→769。
- `zram_patch/002-zstd.patch`：`portability_macros.h` 第 3 个 hunk 因本树
  `DYNAMIC_BMI2` 已改用 `__GNUC__ >= 11` 而拒绝，按实际内容重制 hunk。
- 所有 `|| true` 改为失败即中止，并加产物断言（新源文件存在、旧
  `lz4_compress.c` 已移除、armv8 汇编接入、版本宏）。

**验证**：干净源码树上 `git apply` 干净、0 个 `.rej`、产物含 `1.10.0` / `1.5.7`。

### 4.3 SUSFS 补丁的 task_mmu.c hunk#6 失败（无害）

**现象**

```
Hunk #6 FAILED at 1057.
1 out of 9 hunks FAILED -- saving rejects to file fs/proc/task_mmu.c.rej
```

**判定**：本源码树该段已自带花括号，而 **hunk#7 应用的是同一逻辑的完整版本**
（含 `SUSFS_IS_INODE_SUS_MAP` 判断与 `last_vma_end` 更新）。实测确认
hunk#7 写入的代码块完整覆盖 hunk#6 的语义。

**处理**：允许该单一 reject，但加断言——若 `task_mmu.c` 中**没有**
`SUSFS_IS_INODE_SUS_MAP` 则中止构建；其余任何 `.rej` 一律中止。

### 4.4 `set -e` 与 grep 的陷阱

**现象**：只打印了 reject 文件名就 `exit 1`，走不到「已知无害」分支。

**根因**：

```sh
UNEXPECTED=$(printf '%s\n' "$REJ_FILES" | grep -v '^$' | grep -v '\.rej$')
```

`grep` 无匹配时返回 1，GitHub Actions 默认 `bash -e` 会在此行直接退出。

**处理**：`... | grep -v ... || true` 兜住返回码；断言改用
`if ! grep -q ...; then` 形式，避免 `|| { }` 在 `set -e` 下的同类问题。

### 4.5 heredoc 破坏 YAML 块标量

**现象**：`yaml.safe_load` 报 `ScannerError: could not find expected ':'`。

**根因**：`cat > file <<'PY'` 的终止符 `PY` 写在 YAML 块标量的第 1 列，
把整个 workflow 的缩进结构截断了。

**处理**：改为 base64 传递脚本：

```yaml
run: |
  echo <base64> | base64 -d > /tmp/verify_kpimg.py
  python3 /tmp/verify_kpimg.py
```

> 任何在 `run: |` 内嵌多行脚本的地方都要注意这一点。

### 4.6 管理器没有 KPM 界面

**现象**：BakaSU `v4.2.0-rc3` 的管理器里找不到 KPM 管理。

**证据**：

```sh
# 统计各 APK dex 里的 kpm 字符串数
bakasu.apk  (com.resukisu.resukisu) -> 0     # 只有 KPATCH_NEXT 枚举名
ksunext.apk (com.rifsxd.ksunext)    -> 0
sukisu.apk  (com.sukisu.ultra)      -> 35    # 含 Route.Kpm / kpm load|unload
```

**处理**：`ksu_type=sukisu` 改为真正指向 `SukiSU-Ultra/builtin`（原本被重定向到
BakaSU）。

配套改动：

- **SUSFS 补丁源**改用上游 `simonpunk/susfs4ksu`（`gki-android15-6.6`）。
  builtin 的 `kernel/Kbuild` 依赖 `ksu_install_su_fd` / `ksu_selinux_hide_running`
  等最新钩子，`susfs4oki` 是给 BakaSU 的旧快照，钩子集不匹配。
  已逐符号核对：上游补丁的 11 个 `ksu_*` 钩子在 builtin 中全部存在。
- **`CONFIG_HEADERS_INSTALL=y`**：builtin 的 `kernel/Makefile:25` 用 wildcard
  判断 `usr/include/linux/unistd.h` 是否存在来决定是否编译 `preinit/*.o`，
  关掉会链接期 `undefined symbol: ksu_setup_preinit_file`。
- **`CONFIG_KPM=y`**：builtin 的 `CONFIG_KPM` 默认 `n`。
- **SUSFS 配置项按分支区分**：`HAS_MAGIC_MOUNT` / `AUTO_ADD_SUS_*` /
  `TRY_UMOUNT` 这 5 项只存在于 BakaSU 的 Kconfig，写入 builtin 会被静默忽略。

### 4.7 kpimg 来源错误导致 KPM 名存实亡

**现象**：KPM 界面出现了，但加载模块报「内核不支持 KPM」。

**根因**：SukiSU builtin 的 `sukisu_kpm_*` 是**空桩**（`pr_info` + `asm("nop")`），
必须由 kpimg 用 inline hook 换成真实实现。但两个 kpimg 完全不同：

```sh
strings kpimg-linux   | grep -c sukisu_kpm   # KPatch-Next  -> 0
strings kpimg_sukisu  | grep -c sukisu_kpm   # SukiSU fork  -> 19（7 桩全覆盖）
```

KPatch-Next 的 kpimg hook 的是 `pid_vnr` / `kallsyms_lookup_name` 等通用符号，
跟 `sukisu_kpm_*` 无关。

**处理**：kpimg / kptools 改从
`SukiSU-Ultra/SukiSU_KernelPatch_patch` 的 latest release 获取；
并加前置断言——kpimg 自身必须包含 `sukisu_kpm_version` 字符串。

**验证**：CI 日志 `kpimg 嵌入: 0x236a000 diff 111/294144 (0.038%)`。

### 4.8 内置 KPM 与外挂 KPatch-Next 互斥

`sukisu` 分支不再打包 `kpn.zip`：kpimg 内置注入与 KPatch-Next 模块是两套
KPM 加载器，同时存在会冲突。

### 4.9 版本串伪装

见 §1.1。改 `KERNEL_NAME` + 三个 `KBUILD_BUILD_*` 变量即可，已实测产物 banner
与原厂串**逐字节一致**。

三个坑：

1. `KBUILD_BUILD_TIMESTAMP` **不做日期解析**，由 `init/Makefile` 原样拼入
   `UTS_VERSION`（`cut -b -64`，43 字符未超限）。必须写成 `date(1)` 默认格式；
   写 ISO 格式会原样出现在版本串里。
2. 个位日期是**空格左填充**：`Apr  8` 之间是两个空格，只写一个就不符。
3. 旧方案用 `LD_PRELOAD` 劫持 `date()` **不生效**——`make` 直接展开变量，
   不走被劫持的 libc 调用。

### 4.10 避免 Git 授权弹窗

Git Credential Manager 在多账号环境下会弹窗要求选择账号。用文件型 helper 绕开：

```sh
cat > /tmp/opencode/ghcred.sh <<'EOF'
#!/bin/sh
case "$1" in
  get) printf 'username=StudyAuto\npassword=%s\n' "$(cat /tmp/opencode/.ghtok)" ;;
  *) exit 0 ;;
esac
EOF
chmod 700 /tmp/opencode/ghcred.sh

git config --global credential.helper /tmp/opencode/ghcred.sh
git config --global credential.interactive false
```

令牌取自 `printf "protocol=https\nhost=github.com\n\n" | git credential fill`。

---

## 5. 刷入与安装

### 5.1 刷入内核

> ⚠️ **先备份 boot**（KernelFlasher 等），刷写不可逆。

1. 用 HorizonKernelFlasher / Recovery 刷入 `AnyKernel3_*.zip`
2. 重启

### 5.2 安装管理器

内核 `apk_sign_keys[]` 含 7 个证书，`sig_fallback` 会按证书自动认领管理器，
**无需** `ksud manager add`：

| 管理器 | 包名 | 证书 SHA-256 |
|---|---|---|
| **SukiSU Ultra**（推荐） | `com.sukisu.ultra` | `947ae944f3de4ed4c21a7e4f7953ecf351bfa2b36239da37a34111ad29993eef` |
| KernelSU Next | `com.rifsxd.ksunext` | `79e590113c4c4c0c222978e413a5faa801666957b1212a328e46c00c69821bf7` |
| BakaSU | `com.resukisu.resukisu` | `d3469712b6214462764a1d8d3e5cbe1d6819a0b629791b9f4101867821f1df64` |

安装：

```sh
ADB=/mnt/c/Windows/System32/adb.exe
$ADB push sukisu.apk /data/local/tmp/
$ADB shell pm install -r -g /data/local/tmp/sukisu.apk
$ADB shell monkey -p com.sukisu.ultra -c android.intent.category.LAUNCHER 1
```

**⚠️ 包名必须与内核签名表一致**。用 apktool 改包名会破坏 native 层 JNI 符号
（`libkernelsu.so` 里写死 `Java_com_sukisu_ultra_*`），启动即
`UnsatisfiedLinkError` 闪退，重启不能修复。

### 5.3 刷入后自检

```sh
ADB=/mnt/c/Windows/System32/adb.exe
$ADB shell cat /proc/version                  # 应与原厂串一致
$ADB shell uname -r                           # release 段
$ADB shell su -c 'ksud kpm version'           # 应返回非空版本号
$ADB shell su -c 'ksud module list'
```

---

## 6. SUSFS 模块（SUSFS-FOR-KERNELSU）

### 6.1 不能删除

SUSFS 是「内核提供接口、模块下发配置」的架构：

| 功能 | 提供方 | 删模块后 |
|---|---|---|
| SUS_PATH 隐藏路径 | 模块调 `ksu_susfs add_sus_path` | ❌ 失效 |
| SUS_MOUNT 隐藏挂载 | 模块 | ❌ 失效 |
| SUS_KSTAT 伪装 stat | 模块 | ❌ 失效 |
| SUS_MAP 隐藏映射 | 模块 | ❌ 失效 |
| OPEN_REDIRECT 重定向 | 模块 | ❌ 失效 |
| 版本号伪装 | **已由内核构建提供** | ✅ 不受影响 |

内核只提供能力；具体隐藏哪个路径、哪次挂载，都要模块在开机时下发。

### 6.2 版本伪装已重复

内核已让 `uname(2)` 和 `/proc/version` 都返回原厂串，模块里这一项就多余了：

```sh
/data/adb/susfs4ksu/config.sh
  spoof_uname=1        # 建议改 0
  kernel_version='default'
```

> SUSFS 的 `spoof_uname` 只改 `uname(2)`（补丁只动
> `SYSCALL_DEFINE1(newuname)`），**不改 `/proc/version`**。这是必须靠重编内核
> 才能覆盖的检测点。

### 6.3 `susfsd support` 报 Unsupported（不影响功能）

```
ksu_susfs --help  ->  Detected kernel: susfs v2.3.0 (ABI: sys_reboot)
susfsd support    ->  Unsupported
```

模块自带的 `susfsd` 是按 **KSU ioctl ABI** 编译的，而本内核走 **sys_reboot ABI**。
只影响模块的自动探测路径；`post-fs-data.sh` 有 `dmesg | grep "susfs:"` 兜底，
模块照常初始化。

`susfs-bin-check.sh` 输出 `mismatch` 只是联网比对二进制哈希，与功能无关。

### 6.4 设置版本伪装（如仍想在管理器里操作）

入口：首页 → 底部导航 **SuSFS** → **内核版本伪装**

| 字段 | 值 |
|---|---|
| 内核版本号 | `6.6.118-android15-8-g93e223c276e7-abogki500782043-4k` |
| 内核编译信息 | `#1 SMP PREEMPT Wed Apr  8 15:08:30 UTC 2026` |

点「应用」即时生效；打开 **Auto Start** 才能开机自动应用。

---

## 7. 仓库结构

```
.github/workflows/fastbuild_6.6.118.yml   本机型构建流程（唯一改造过的）
zram_patch/001-lz4.patch                  lz4 1.10.0（已按本树重制）
zram_patch/002-zstd.patch                 zstd 1.5.7（已按本树重制）
other_patch/                              69_hide_stuff / cve / apk_sign 等
droidspaces_patch/                        容器支持补丁
local/builder_*.sh                        上游本地脚本（未改造）
```

相对上游 `cctv18/oppo_oplus_realme_sm8750` 的改动集中在
`fastbuild_6.6.118.yml`，共 8 组（见 §4）。

---

## 8. 常见问题速查

| 现象 | 原因 | 处理 |
|---|---|---|
| `Please wait susfs update.` | BakaSU main 无条件 `$(error)` | 用 `sukisu` 分支，或锁 `v4.2.0-rc3` |
| 构建成功但没 lz4 | `git apply` 原子回滚 + `|| true` | 已修，见 §4.2 |
| 只打印 rej 就 exit 1 | grep 返回 1 + `set -e` | `grep ... \|\| true` |
| YAML `ScannerError` | heredoc 终止符顶格 | base64 传脚本 |
| KPM 界面报不支持 | kpimg 用错 | 用 SukiSU 官方 kpimg |
| `undefined symbol: ksu_setup_preinit_file` | builtin 未开 `HEADERS_INSTALL` | 设 `=y` |
| 管理器闪退 | 改包名破坏 JNI 符号 | 保持官方包名 |
| `/proc/version` 还是真实值 | SUSFS 只改 `uname(2)` | 重编内核做伪装 |
| `--depth=1` 后版本号异常 | `rev-list --count` 失真 | 不要对 KSU 仓库用浅克隆 |
| `CONFIG_REKERNEL` 无效 | 本源码树无该符号 | 保持关闭 |

---

## 9. 产物命名

```
AnyKernel3_<KSU>_<KSUVER>_6.6_<KERNEL_NAME>.zip
```

本次成品：

```
AnyKernel3_SukiSU_35363_6.6_android15-8-g93e223c276e7-abogki500782043-4k.zip
```

> 版本串伪装后，**无法再从名字区分自编内核与原厂**。识别自编版靠
> `KSUVER`（本次 `35363`）与 KPM/SUSFS 配置项。
