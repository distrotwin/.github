# distrotwin · 国产桌面操作系统的构建与测试镜像

把国产桌面操作系统做成可用于**软件构建、打包与兼容性测试**的容器镜像，公开在 GHCR，匿名可拉。

## 为什么做这个

给国产桌面 OS 交付软件的人都会遇到同一个问题：CI 里没有对应的环境。我们实测了公开名录里 42 个候选容器镜像引用，又直接枚举了麒麟软件与统信自有 registry 的全部公开项目——**主流商业国产桌面 OS 没有一家提供官方的桌面线容器镜像**。缺了它，编译要么在真机上手工做，要么拿一个血统相近的社区镜像凑合：前者进不了 CI，后者的 glibc/libstdc++ 基线跟真实目标对不上，验出来的结论不可信。

这个 org 对着厂商官方发布的介质或软件源（ISO、公开 apt/rpm 源），按依赖闭包自举出 micro / base / devel 三档镜像。每档带构建期与镜像内两层校验，发布前跑数百项验收；镜像 label 里记录可追溯到厂商锚点（ISO 官方校验和、源 Release 指纹、GPG key 指纹）的证据链。

边界要说清：**镜像里没有内核**——所有 Linux 容器都共享宿主内核，依赖内核态的安全机制不生效，`systemd` 有二进制但不是 PID 1。所以它适合回答「**编出来的东西对不对**」：产物需要的 glibc/libstdc++ 符号版本目标系统能否满足、依赖闭包全不全、编译问题能不能在这个系统上复现。它不适合当生产运行时底座，也替代不了整机系统验收。

## 一分钟上手（以银河麒麟为例）

```bash
docker run -it --rm ghcr.io/distrotwin/kylin:v10sp1-devel /bin/bash
```

```bash
echo '#include <stdio.h>
int main(void){ int a, b; if (scanf("%d %d", &a, &b) != 2) return 1; printf("%d\n", a + b); return 0; }' > ab.c

gcc -O2 -o ab ab.c
echo "3 4" | ./ab
objdump -T ab | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1
```

最后一行直接告诉你**产物要求目标系统多新的 glibc**——在 V10 SP1 上不该超过 `GLIBC_2.31`。换任何一个受支持的系统只是换 tag，各仓库 README 有完整说明与基线数字。

## 国产桌面 OS 名录与镜像现状

一行一个系统，版本与状态截至 2026-09。现状：🟢 维护中 · 🟡 状态存疑 · ⚫ 停滞或已并入。如何使用：✅ 用本 org 的镜像 · 📦 官方已有镜像（引用均经匿名拉取实测）· ⏳ 待定 · 🚫 不做（附原因）。

| OS | 最新版本（发布时间） | 现状 | 如何使用 |
|---|---|---|---|
| 银河麒麟桌面 | V11（2025-08） | 🟢 在售维护，安可公告入围 | ✅ [`kylin`](https://github.com/distrotwin/kylin)：V4 / V10 / V10 SP1 / V11，V11 含 LoongArch |
| 统信 UOS 桌面 | V25（2026-04） | 🟢 在售维护，安可公告入围 | ✅ [`uos`](https://github.com/distrotwin/uos)：V20 / V25（官方 ISO 切片），V25 含 LoongArch |
| 麒麟信安桌面 | V6 SP1（2026-06） | 🟢 在售维护（电力行业） | ✅ [`kylinsec`](https://github.com/distrotwin/kylinsec)：V6（含 LoongArch）/ V3.4 |
| 凝思安全操作系统 | 多支并行，最新构建 2026-06 | 🟢 在售维护（电力调度） | ✅ [`linx`](https://github.com/distrotwin/linx)：六支全谱系，glibc 2.5→2.38 |
| 方德桌面 | V5.0（2022-06；Pro 版 2025-09 过安可） | 🟢 在售维护，安可公告入围 | ✅ [`fangde`](https://github.com/distrotwin/fangde)：公开源三代（v3.1 / panda / tiger） |
| Loongnix 桌面 | 25.1（2026-08） | 🟢 社区活跃（龙芯官方社区版） | ✅ [`loongnix`](https://github.com/distrotwin/loongnix)：25 线 loong64（glibc 2.41）；20 线是 LoongArch 旧世界 ABI，通用模拟器跑不起来，未收录 |
| openKylin | 3.0（2026-08） | 🟢 社区活跃 | 📦 `docker pull openkylin/openkylin:3.0`，官方已覆盖 |
| deepin | 25.2.1（2026-08） | 🟢 社区活跃 | 📦 `docker pull linuxdeepin/deepin`，官方已覆盖 |
| openEuler | 24.03 LTS SP4（2026-06） | 🟢 社区活跃（服务器系，桌面以软件包提供） | 📦 `docker pull openeuler/openeuler:24.03-lts`，另有 DevStation 官方 rootfs tar |
| Ubuntu Kylin（优麒麟） | 26.04.1 LTS（2026-08） | 🟢 社区活跃 | 🚫 不做：底座是 Ubuntu 直系，官方 `ubuntu` 镜像即基本同质，自建没有 ABI 增量 |
| Anolis OS（龙蜥） | 23.5 | 🟢 社区活跃（服务器定位） | 📦 服务器镜像官方已有（`openanolis/anolisos`）；桌面无独立产品线，桌面向不做 |
| OpenCloudOS | 9.6（2026-07） | 🟢 社区活跃（服务器定位） | 📦 服务器镜像官方已有（`opencloudos/opencloudos:9.0`）；桌面无独立 ISO，桌面向不做 |
| AOSC OS（安同） | 滚动（2026-06 快照） | 🟢 社区活跃 | 🚫 不做：滚动发行加自研包管理 oma，不适合做成固定版本快照；官方 rootfs tarball 可直接 `docker import` 使用 |
| RevyOS | 20260504 快照 | 🟢 社区活跃（RISC-V） | 🚫 不做：定位是 RISC-V 开发板系统，不在桌面构建测试范围 |
| Bianbu OS | v4.0.6（2026-08） | 🟢 在售（RISC-V，进迭时空） | 🚫 不做：同上，RISC-V 开发板向 |
| 新支点桌面（NSDL） | V4.5.2（官方未标日期） | 🟢 在售（中兴系） | ⏳ 待定：官网未见公开 ISO/源直链，材料可得性尚未实测；探明有公开材料即可接入 |
| 普华桌面 | 官方两处版本不一致（V4.0 / V5.0） | 🟡 状态存疑 | 🚫 不做：官方版本信息不一致，且无公开介质可验 |
| 中标麒麟桌面（NeoKylin） | V7（历史产品） | ⚫ 品牌已并入麒麟软件 | 🚫 不做：产品线由银河麒麟延续，已被 [`kylin`](https://github.com/distrotwin/kylin) 覆盖 |
| 一铭桌面 | 4.0 SP1（2016-06） | ⚫ 停滞（2016 后无公开更新） | 🚫 不做：材料不可得 |
| FydeOS | v23 | 🟢 在售 | 🚫 不做：ChromiumOS 用户态，常规 Linux 容器方式装不出等价环境 |
| EulerOS（华为） | 未公开 | 🟢 商业在售（服务器） | 🚫 不做：无桌面产品线，也无公开下载 |

## 本 org 的仓库

| 仓库 | 作用 |
|---|---|
| [`buildkit`](https://github.com/distrotwin/buildkit) | 公共构建机器码：五条构建路径、数据镜像取材、测试与验收门禁、可复用 CI workflow；机制文档与已知问题记录在 `docs/` |
| [`kylin`](https://github.com/distrotwin/kylin) | 银河麒麟 V4 / V10 / V10 SP1 / V11 → [镜像](https://github.com/distrotwin/kylin/pkgs/container/kylin) |
| [`uos`](https://github.com/distrotwin/uos) | 统信 UOS V20 / V25 → [镜像](https://github.com/distrotwin/uos/pkgs/container/uos) |
| [`kylinsec`](https://github.com/distrotwin/kylinsec) | 麒麟信安 V6 / V3.4 → [镜像](https://github.com/distrotwin/kylinsec/pkgs/container/kylinsec) |
| [`loongnix`](https://github.com/distrotwin/loongnix) | Loongnix 桌面 25（loong64） → [镜像](https://github.com/distrotwin/loongnix/pkgs/container/loongnix) |
| [`linx`](https://github.com/distrotwin/linx) | 凝思六支全谱系（glibc 2.5→2.38） → [镜像](https://github.com/distrotwin/linx/pkgs/container/linx) |
| [`fangde`](https://github.com/distrotwin/fangde) | 方德桌面三代（v3.1 / panda / tiger） → [镜像](https://github.com/distrotwin/fangde/pkgs/container/fangde) |
| `scratch` | 数据镜像：厂商站点对 CI 不可达时的介质中转，一个介质一个 tag，带完整性锚点 |

每个镜像三档：`micro`（最小根系统）/ `base`（常用工具）/ `devel`（编译工具链）。tag 规则、基线数字与各系统的已知怪癖见对应仓库的 README。
