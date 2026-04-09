# rootAVD Python 改写计划

## 目标

将 rootAVD 的宿主端逻辑（当前由 `rootAVD.bat` + `rootAVD.sh` 角色A 共同承担）改写为一个跨平台的 `rootAVD.py`，同时将 `rootAVD.sh` 精简为只包含 AVD 内部打补丁逻辑的 `rootAVD_patch.sh`。

## 现状架构

```
rootAVD.bat (Windows 宿主端)
    ├── 参数解析 (:ProcessArguments)
    ├── SDK 路径查找 (:GetANDROIDHOME)
    ├── ADB 连接测试 (:TestADB, :TestADBWORKDIR)
    ├── adb push/pull (:pushtoAVD, :pullfromAVD)
    ├── 备份/恢复 (:create_backup, :restore_backups)
    ├── APK 安装 (:installapps)
    ├── 关机 (:ShutDownAVD)
    └── adb shell sh rootAVD.sh  ──→  AVD 内执行

rootAVD.sh (双重角色)
    ├── 角色A: Mac/Linux 宿主端 (等价于 .bat)
    │   ├── ProcessArguments()
    │   ├── GetANDROIDHOME()
    │   ├── TestADB()
    │   ├── CopyMagiskToAVD()  ← 核心入口：push/pull/install
    │   ├── pushtoAVD() / pullfromAVD()
    │   ├── create_backup() / restore_backups()
    │   ├── install_apps()
    │   └── ShutDownAVD()
    │
    └── 角色B: AVD 内部打补丁引擎
        ├── api_level_arch_detect()
        ├── PrepBusyBoxAndMagisk()  ← 解压 Magisk.zip, 找 BusyBox
        ├── ExecBusyBoxAsh()        ← 在 BusyBox ASH standalone 下重新执行
        ├── InstallMagiskToAVD()    ← 总调度
        │   ├── get_flags()
        │   ├── copyARCHfiles()
        │   ├── detect_ramdisk_compression_method()
        │   ├── decompress_ramdisk()       ← 处理 dual-cpio
        │   ├── process_fake_boot_img()    ← 自动 boot_patch.sh 流程
        │   │   ├── create_fake_boot_img()
        │   │   ├── run_boot_patch_sh()
        │   │   └── unpack_new_boot_img()
        │   ├── test_ramdisk_patch_status()
        │   ├── verify_ramdisk_origin()
        │   ├── apply_ramdisk_hacks()      ← fstab/rc overlay
        │   ├── patching_ramdisk()         ← fallback 手动注入
        │   ├── repacking_ramdisk()
        │   └── rename_copy_magisk()
        │
        ├── CheckAvailableMagisks()  ← 在线选择 Magisk 版本
        ├── DownLoadFile()
        └── update_lib_modules()     ← 内核模块处理
```

## 目标架构

```
rootAVD.py (跨平台宿主端，替代 .bat 和 .sh 角色A)
    ├── 参数解析 (argparse)
    ├── SDK 路径查找 (ANDROID_HOME / 默认路径)
    ├── ADB 连接管理 (subprocess 调用 adb)
    ├── 文件推送/拉取 (adb push/pull)
    ├── 备份/恢复
    ├── APK 安装
    ├── 关机
    └── adb shell sh rootAVD_patch.sh  ──→  AVD 内执行

rootAVD_patch.sh (AVD 内部打补丁引擎，只保留角色B)
    └── 纯 Android shell 环境下运行的打补丁逻辑
```

## 分阶段改写步骤

### Phase 1: 创建 rootAVD.py 宿主端框架

**新建 `rootAVD.py`**，实现以下模块：

#### 1.1 ADB 工具类
从 rootAVD.bat 和 rootAVD.sh 中迁移：
- `TestADB()` / `:TestADB` → `ADB.find_adb()`, `ADB.test_connection()`
- `pushtoAVD()` / `:pushtoAVD` → `ADB.push()`
- `pullfromAVD()` / `:pullfromAVD` → `ADB.pull()`
- `ShutDownAVD()` / `:ShutDownAVD` → `ADB.shutdown()`
- adb shell 命令封装 → `ADB.shell()`

#### 1.2 SDK 路径查找
从 rootAVD.bat `:GetANDROIDHOME` 和 rootAVD.sh `GetANDROIDHOME()` 迁移：
- 自动检测 ANDROID_HOME 环境变量
- 平台默认路径：
  - Windows: `%LOCALAPPDATA%\Android\Sdk`
  - macOS: `~/Library/Android/sdk`
  - Linux: `~/Android/Sdk`
- 枚举 system-images 目录

#### 1.3 参数解析
从 rootAVD.bat `:ProcessArguments` 和 rootAVD.sh `ProcessArguments()` 迁移：
- 使用 `argparse` 实现
- 支持的参数：ramdisk 路径, restore, DEBUG, PATCHFSTAB, GetUSBHPmodZ,
  InstallKernelModules, InstallPrebuiltKernelModules, ListAllAVDs, InstallApps, AddRCscripts

#### 1.4 备份/恢复
从两端 `create_backup()` / `restore_backups()` 迁移：
- 备份 ramdisk.img → ramdisk.img.backup
- restore 模式恢复所有 .backup 文件

#### 1.5 APK 安装
从 rootAVD.bat `:installapps` 和 rootAVD.sh `install_apps()` 迁移：
- 遍历 Apps/ 目录安装所有 APK
- 处理 INSTALL_FAILED_UPDATE_INCOMPATIBLE 自动卸载重装

#### 1.6 主流程编排
从 rootAVD.bat 主逻辑 和 rootAVD.sh `CopyMagiskToAVD()` 迁移：
```python
def main():
    args = parse_args()
    sdk = find_sdk()
    adb = ADB(sdk)
    adb.test_connection()

    if args.restore:
        restore_backups(args.ramdisk_path)
        return

    if args.install_apps:
        install_apps(adb)
        return

    if args.list_all:
        list_avds(sdk)
        return

    # 备份
    create_backup(args.ramdisk_path)

    # 推送
    adb.shell("rm -rf {workdir}/Magisk")
    adb.shell("mkdir {workdir}/Magisk")
    adb.push("Magisk.zip", "{workdir}/Magisk/")
    adb.push(args.ramdisk_path, "{workdir}/Magisk/ramdisk.img")
    adb.push("rootAVD_patch.sh", "{workdir}/Magisk/")

    # 执行打补丁
    adb.shell("sh {workdir}/Magisk/rootAVD_patch.sh", args=original_args)

    # 拉取结果
    adb.pull("{workdir}/Magisk/ramdiskpatched4AVD.img", args.ramdisk_path)
    adb.pull("{workdir}/Magisk/Magisk.apk", "Apps/")

    # 清理、安装、关机
    adb.shell("rm -rf {workdir}/Magisk")
    install_apps(adb)
    adb.shutdown()
```

### Phase 2: 拆分 rootAVD.sh → rootAVD_patch.sh

**从 rootAVD.sh 中剥离宿主端代码，只保留 AVD 内部逻辑。**

#### 2.1 删除的函数（宿主端，已迁移到 Python）
| 函数 | 行号 | 说明 |
|---|---|---|
| `TestADB()` | 573 | ADB 连接测试 |
| `CopyMagiskToAVD()` | 685 | 宿主端主流程 |
| `pushtoAVD()` | 469 | adb push 封装 |
| `pullfromAVD()` | 486 | adb pull 封装 |
| `create_backup()` | 499 | 文件备份 |
| `restore_backups()` | 520 | 恢复备份 |
| `toggle_Ramdisk()` | 541 | 切换 ramdisk |
| `install_apps()` | 439 | APK 安装 |
| `ShutDownAVD()` | 641 | 关机 |
| `GetANDROIDHOME()` | 2601 | SDK 路径查找 |
| `GetAVDPKGRevision()` | 672 | 读取 source.properties |
| `FindSystemImages()` | 2662 | 枚举 system-images |
| `ShowHelpText()` | 2711 | 帮助文本 |
| `ProcessArguments()` | 2785 | 参数解析 |
| `checkfile()` | 418 | 文件检查辅助 |

#### 2.2 保留的函数（AVD 内部打补丁引擎）
| 函数 | 说明 |
|---|---|
| `getdir()` | 路径辅助 |
| `get_flags()` | 检测 KEEPVERITY 等标志 |
| `api_level_arch_detect()` | 架构和 API 检测 |
| `copyARCHfiles()` | 复制架构相关文件 |
| `abort_script()` | 错误退出 |
| `compression_method()` | 检测压缩格式 |
| `detect_ramdisk_compression_method()` | 检测 ramdisk 压缩 |
| `run_boot_patch_sh()` | 自动调用 boot_patch.sh |
| `unpack_new_boot_img()` | 解包 new-boot.img |
| `create_fake_boot_img()` | 创建 fake boot.img |
| `process_fake_boot_img()` | fake boot.img 总流程 |
| `decompress_ramdisk()` | 解压 ramdisk (处理 dual-cpio) |
| `repack_ramdisk()` | 重打包 ramdisk |
| `test_ramdisk_patch_status()` | 检测 ramdisk 打补丁状态 |
| `verify_ramdisk_origin()` | 验证 ramdisk 来源 |
| `apply_ramdisk_hacks()` | fstab/rc overlay |
| `patching_ramdisk()` | fallback 手动注入 Magisk |
| `repacking_ramdisk()` | 压缩回 ramdisk.img |
| `rename_copy_magisk()` | Magisk.zip → Magisk.apk |
| `update_lib_modules()` | 内核模块处理 |
| `PrepBusyBoxAndMagisk()` | 解压 Magisk.zip, 找 BusyBox |
| `ExecBusyBoxAsh()` | BusyBox ASH standalone 重执行 |
| `FindUnzip()` | 找 unzip 解压 Magisk |
| `FindWorkingBusyBox()` | 找可用 BusyBox |
| `TestingBusyBoxVersion()` | 测试 BusyBox |
| `CopyBusyBox()` / `MoveBusyBox()` | BusyBox 复制/移动 |
| `CheckAvailableMagisks()` | 在线 Magisk 版本选择 |
| `DownLoadFile()` | 下载文件 |
| `CheckAVDIsOnline()` | 检测 AVD 网络 |
| `InstallMagiskToAVD()` | AVD 内部总调度 |
| BlueStacks 相关函数 | 如需保留 BlueStacks 支持 |

#### 2.3 简化入口
rootAVD_patch.sh 的入口简化为：
```bash
# 脚本只在 AVD 内部运行
SHELLRESULT=$(getprop 2>/dev/null)
if [[ "$?" != "0" ]]; then
    echo "[!] This script must run inside an Android AVD"
    exit 1
fi
InstallMagiskToAVD $@
```

### Phase 3: 删除 rootAVD.bat

rootAVD.py 完全替代 rootAVD.bat 的功能后，删除 rootAVD.bat。

### Phase 4: 保留 rootAVD.sh 作为兼容入口 (可选)

如果希望保持向后兼容，可以保留一个极简的 rootAVD.sh 作为 wrapper：
```bash
#!/usr/bin/env bash
# Legacy wrapper - calls rootAVD.py
python3 "$(dirname "$0")/rootAVD.py" "$@"
```

## 文件结构（改写后）

```
rootAVD/
├── rootAVD.py              # 跨平台宿主端 (替代 .bat 和 .sh 角色A)
├── rootAVD_patch.sh        # AVD 内部打补丁引擎 (原 .sh 角色B)
├── rootAVD.sh              # (可选) 兼容 wrapper → 调用 rootAVD.py
├── Magisk.zip              # Magisk 安装包
├── Apps/                   # APK 目录
│   └── Magisk.apk
├── README.md
├── REFACTOR_PLAN.md        # 本文件
├── .gitignore
└── .gitattributes
```

## 注意事项

1. **Python 版本**: 使用 Python 3.6+，仅用标准库 (subprocess, argparse, pathlib, os, platform)，不引入第三方依赖
2. **ADB 路径**: 优先 PATH 中的 adb，其次 SDK 目录下的 platform-tools/adb
3. **编码**: 处理 adb 输出时注意 Windows 上的 GBK 编码
4. **BlueStacks**: 如果不需要 BlueStacks 支持，可以在 Phase 2 中一并移除相关函数，大幅减少 rootAVD_patch.sh 的体积
5. **Magisk 版本选择菜单**: 当前在 AVD 内部通过 BusyBox wget 下载 JSON 实现，改写后可考虑移到 Python 端（宿主端有更好的网络环境），下载完再 push 进去
6. **向后兼容**: 保留原始的命令行参数格式，用户无需改变使用习惯

---

# boot_patch.sh 自动流程合入计划

## 背景

本分支 (`dev/auto-boot-patch`) 基于 `origin/master`（上游 newbit1）开发，但 `nku100/master`（我们的 fork）已在上游基础上有 14 个额外提交，包含 init-ld 支持、BusyBox 检测放宽、PREINITDEVICE 等改进。

经测试，**nku100/master 已经可以正常 root Android 37 ps16k AVD**。

因此需要：以 `nku100/master` 为基准，仅合入本分支中独有的有价值改动。

## 两边改动对比

### 重复部分（不合入，nku100/master 已有）

| 改动 | nku100/master 实现 | 本分支实现 | 结论 |
|---|---|---|---|
| **init-ld 支持** | `$INITLD` 标志 + `$SKIPLD` 注释行控制新旧格式共存于同一个 cpio 命令 | `if [ -e magisk ]` 分两套完整 if/else 代码块 | **用 nku100**，更简洁 |
| **BusyBox 检测放宽** | `*"BusyBox"*` （最宽松） | `*"BusyBox"*"multi-call"*` （去掉了 Magisk，保留 multi-call） | **用 nku100**，更兼容 |
| **PREINITDEVICE** | ✅ 在 `patching_ramdisk()` 中通过 `magisk --preinit-device` 获取并写入 config | ❌ fallback 路径中遗漏（但默认走 boot_patch.sh 自带处理） | **用 nku100**，更完整 |
| **Magisk v30.7** | ✅ Magisk.zip 已是 v30.7 | ✅ 同 | 相同 |

### 独有部分（需要合入）

| 改动 | 说明 | 涉及函数/文件 |
|---|---|---|
| **boot_patch.sh 自动打补丁** | 调用 Magisk 官方 `boot_patch.sh` 替代手动 UI 交互和手动 cpio 注入 | 新增 `run_boot_patch_sh()`, `unpack_new_boot_img()` |
| **process_fake_boot_img() 改写** | 自动流程：`create_fake_boot_img` → `run_boot_patch_sh` → `unpack_new_boot_img` | 修改 `process_fake_boot_img()` |
| **create_fake_boot_img() 精简** | 移除末尾的 `InstallMagiskTemporarily` / `detecting_users` / `runMagisk_to_Patch_fake_boot_img` / `RemoveTemporarilyMagisk` 调用 | 修改 `create_fake_boot_img()` |
| **无条件执行 process_fake_boot_img** | `InstallMagiskToAVD()` 中去掉 `if $FAKEBOOTIMG` 判断 | 修改 `InstallMagiskToAVD()` |
| **FAKEBOOTIMG 死代码清理** | 移除标志定义、解析、导出、debug 打印 | `ProcessArguments()` 等 |
| **废弃函数删除** | 删除 `runMagisk_to_Patch_fake_boot_img()`, `detecting_users()` | rootAVD.sh |
| **帮助文本清理** | 移除 FAKEBOOTIMG 相关说明和命令示例 | rootAVD.sh, rootAVD.bat, README.md |
| **REFACTOR_PLAN.md** | Python 改写计划文档 | 新文件 |
| **.gitignore** | 新增 `.claude/` | .gitignore |

## 执行步骤

```bash
# 1. 基于 nku100/master 创建新分支
git checkout nku100/master
git checkout -b dev/merge-boot-patch

# 2. 新增两个函数：run_boot_patch_sh() 和 unpack_new_boot_img()
#    插入到 rootAVD.sh 的 detect_ramdisk_compression_method() 之后

# 3. 修改 create_fake_boot_img()
#    删除末尾4行：InstallMagiskTemporarily / detecting_users /
#    runMagisk_to_Patch_fake_boot_img / RemoveTemporarilyMagisk

# 4. 改写 process_fake_boot_img()
#    else 分支：create_fake_boot_img → run_boot_patch_sh → unpack_new_boot_img

# 5. 修改 InstallMagiskToAVD()
#    去掉 `if $FAKEBOOTIMG; then ... fi`，process_fake_boot_img 无条件执行

# 6. 清理 FAKEBOOTIMG 死代码
#    - ProcessArguments(): 删除 FAKEBOOTIMG=false 初始化
#    - ProcessArguments(): 删除 FAKEBOOTIMG 解析 if 块
#    - ProcessArguments(): 删除 export FAKEBOOTIMG
#    - debug 打印: 删除 echo "FAKEBOOTIMG: $FAKEBOOTIMG"
#    - 删除 runMagisk_to_Patch_fake_boot_img() 函数
#    - 删除 detecting_users() 函数

# 7. 清理帮助文本
#    - rootAVD.sh: ShowHelpText 中删除 FAKEBOOTIMG 说明和命令示例
#    - rootAVD.bat: 同上
#    - README.md: 同上，更新 "Fake Boot.img Function" → "Auto Boot Patching"

# 8. 更新 .gitignore 添加 .claude/

# 9. 添加 REFACTOR_PLAN.md

# 10. 提交并推送到 nku100
git add -A
git commit -m "feat: replace FAKEBOOTIMG with automatic boot_patch.sh flow"
git push nku100 dev/merge-boot-patch
```

## 注意事项

- `patching_ramdisk()` **不做改动**，保留 nku100/master 的版本（有 `$INITLD`/`$SKIPLD` 和 PREINITDEVICE），作为 boot_patch.sh 失败时的 fallback
- `TestingBusyBoxVersion()` 中 grep 返回值判断（`255` → `0`）需要确认 nku100/master 是否已修，若未修则一并合入
- `AllowPermissionsTo3rdPartyAPKs()` 和 `MAGISK_CHOICE` 等 nku100 独有功能保持不动
