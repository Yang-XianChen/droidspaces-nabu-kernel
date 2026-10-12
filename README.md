# Droidspaces Kernel Builder — Xiaomi Pad 5 (nabu) / crDroid 12.x OFFICIAL

用 GitHub Actions 从 **crDroid 官方内核源码**出发,自动编出一个**带 Droidspaces 支持、且保留 KernelSU Next + susfs root** 的内核卡刷包。

> 方法论与踩坑记录见姊妹仓库 [droidspaces-psyche-kernel](https://github.com/Yang-XianChen/droidspaces-psyche-kernel)
> (那是小米 12X;本次是 Pad 5,内核树完全不同)。

---

## 这台设备的关键事实(已核实)

| 项 | 值 |
|---|---|
| 设备 | Xiaomi Pad 5,代号 **nabu**,SoC **SM8150**(骁龙 860) |
| ROM | crDroid 12.x,**OFFICIAL + OTA**,用 crDroid 签名密钥 |
| 内核 | **4.14.357**(比 12X 的 4.19 更老) |
| 内核源码 | `crdroidandroid/android_kernel_xiaomi_sm8150` @ `16.0` |
| Root | 内核树**已内置** KernelSU Next + susfs(`drivers/kernelsu/`),Kconfig 里 `KSU` 与 `KSU_SUSFS` 均 `default y`;arm64 `select THREAD_INFO_IN_TASK` 满足 susfs 依赖 |
| 镜像格式 | `Image.gz`(BOARD_KERNEL_IMAGE_NAME,header v3,dtb 在 vendor_boot) |
| defconfig 拼接 | `vendor/sm8150-perf_defconfig` + `vendor/debugfs.config` + `vendor/xiaomi/sm8150-common.config` + `vendor/xiaomi/nabu.config` |

**和 12X 完全同构的硬阻断**:`sm8150-perf_defconfig` 里写着 `# CONFIG_PID_NS is not set`,
且 `SYSVIPC` / `POSIX_MQUEUE` / `DEVTMPFS` 都没开 —— 容器根本起不来。必须自己编内核。

实测:该树 `=m` 条目为 **0**(全部驱动编进内核),所以改配置不会引发模块符号 CRC 不匹配。

---

## ⚠️ 与 12X 最大的不同:这是带 OTA 的官方 ROM

**刷了自定义内核后,crDroid 的 OTA 更新会把 boot 分区整个换回官方内核**,Droidspaces 支持随之消失。
每次 OTA 之后都需要重新刷一遍本仓库产出的包。

如果你更看重"OTA 无感",可以不做这件事;如果你要 Droidspaces,就得接受这个手动步骤。

---

## 一次性搭建

1. GitHub 新建空仓库,例如 `droidspaces-nabu-kernel`(**不要**勾模板)。
2. 按下述结构推上去:

```
droidspaces-nabu-kernel/
├── .github/workflows/build.yml
├── anykernel/anykernel.sh
├── config/nabu-droidspaces.config
└── README.md
```

```bash
git init && git add -A
git commit -m "Droidspaces kernel builder for nabu"
git branch -M main
git remote add origin https://github.com/<你的用户名>/droidspaces-nabu-kernel.git
git push -u origin main
```

---

## 运行构建

**Actions → Build Droidspaces Kernel (Xiaomi Pad 5 / nabu) → Run workflow**

| 输入 | 默认 | 说明 |
|---|---|---|
| `kernel_repo` | `crdroidandroid/android_kernel_xiaomi_sm8150` | 不用改 |
| `kernel_ref` | `16.0` | 也可填日期分支或提交号(见下) |
| `publish_release` | `true` | 构建完发到 Releases |

### 选对版本(建议)

维护者会打**带日期的分支**,正好用来对齐你的 ROM 构建日期:

| 分支 | 说明 |
|---|---|
| `16.0` | 主分支,当前 HEAD 2026-09-17 |
| `16.0-20260426` | 2026-04-26 快照 |
| `16.0-20260204` | 2026-02-04 快照 |
| `16.0-qpr0` | QPR0 基线 |

先在平板上查:`设置 → 关于平板 → 版本号`(形如 `crDroidAndroid-16.0-20260426-nabu-v12.x`),
把日期对应的分支填进 `kernel_ref`。不确定就先用默认的 `16.0` 试。

也可以直接填提交号 —— 工作流对非分支名会走 codeload tarball,即使该提交已被 rebase 掉也能取到。

---

## 刷入

产出是标准 AnyKernel3 卡刷包(**只替换 boot 里的 kernel**,保留 ramdisk / dtb,不动 vendor_boot 与 vendor)。

- **推荐**:平板上用支持 AnyKernel3 的刷入器(有 root 即可,例如 KernelFlasher)。
- 或解出 `Image.gz`,用官方 `boot.img` 重新打包后 `fastboot flash boot`。

**先备份**:官方 ROM 包里的 `boot.img` 就是最好的回滚备份,建议留一份。

---

## 刷完验收

```sh
adb shell cat /proc/version                      # 应是新内核
adb shell su -c id                               # 应返回 uid=0(root 还在)
adb shell 'ls -l /proc/self/ns/'                 # 必须能看到 pid 和 ipc
adb shell 'zcat /proc/config.gz | grep -E "PID_NS|IPC_NS|SYSVIPC|DEVTMPFS"'
```

然后在平板上装 **Droidspaces APK** → 授权 root → 装后端 → 重启 →
**Settings → Requirements → Check Requirements** 全绿。

装容器时发行版**只能用 systemd 258 之前的**:Ubuntu 22.04/24.04、Debian 12/13、
Alpine(OpenRC)、Void、OpenWrt —— Arch / Fedora 这类滚动版起不来(内核缺 `openat2`)。
4.14 内核同样受此限制。

---

## 这个方案改动了什么(透明清单)

1. **追加配置**:把 `config/nabu-droidspaces.config` 追加到 defconfig 拼接末尾,
   合并成 `nabu_ds_defconfig` 使用。核心是打开原本被显式关闭的 `CONFIG_PID_NS`,
   补上 `SYSVIPC` / `POSIX_MQUEUE` / `IPC_NS`(前两者是 `IPC_NS` 的 Kconfig 前置)与 `DEVTMPFS`,
   并按需打开 `USER_NS`、`CGROUP_PIDS`、`MACVLAN`、`TMPFS_XATTR` 等。
2. **打官方 non-GKI 补丁**。实测:
   - `02.fix_restore cgroup file prefix handling` → `kernel/cgroup/cgroup.c` **干净应用**(hunk 偏移 310 行)。
   - `01.fix_kernel_panic_in_xt_qtaguid` → 目标 `net/netfilter/xt_qtaguid.c` **在该树中已被移除**,
     自动跳过并告警(与 12X 相同)。
3. **编译后只取 `Image.gz`** 打进 AnyKernel3,不动 dtb / dtbo / vendor_boot / vendor 分区。
