# distrotwin · 国产桌面操作系统的构建与测试镜像

把国产桌面操作系统做成可用于**软件构建、打包与兼容性测试**的容器镜像，公开在 GHCR，任何人可匿名拉取。

## 为什么做这个

给国产桌面 OS 交付软件的人都会遇到同一个问题：CI 里没有对应的环境。逐个实测过名录里 42 个候选容器镜像引用、并直接枚举了麒麟软件与统信自有 registry 的全部公开项目之后，结论是：**主流商业国产桌面 OS 没有一家提供官方桌面线容器镜像**（麒麟的 `cr.kylinos.cn` 27 个仓库、统信的 `registry.uniontech.com` 18 个公开仓库，含 desktop/ukui/dde 字样的均为零）。于是编译要么在真机上手工做，要么拿一个"血统相近"的社区镜像凑合——前者进不了 CI，后者的 glibc/libstdc++ 基线跟真实目标对不上，验出来的结论不可信。

这个 org 做的事：对着**厂商官方发布的介质或软件源**（ISO、公开 apt/rpm 源），按依赖闭包自举出 micro / base / devel 三档镜像，每一档带构建期与镜像内两层校验，发布前跑数百项验收，镜像 label 里记录可追溯到厂商锚点（ISO 官方校验和、源 Release 指纹、GPG key 指纹）的完整证据链。

定位边界同样重要：**镜像里没有内核**（所有 Linux 容器共享宿主内核），依赖内核态的安全机制不生效，`systemd` 有二进制但不是 PID 1。所以它适合回答「**编出来的东西对不对**」——产物需要的 glibc/libstdc++ 符号版本目标系统能否满足、依赖闭包全不全、这个系统上能不能复现编译问题；不适合回答「跑起来的系统对不对」，不能当生产运行时底座或系统验收替代品。

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

最后一行是这套镜像最有用的一句：它直接告诉你**产物要求目标系统多新的 glibc**——在 V10 SP1 上不该超过 `GLIBC_2.31`。换任何一个受支持的系统，只是换 tag 的事（各仓库 README 有该系统的完整说明与基线数字）。

## 国产桌面 OS 名录与镜像现状

一行一个系统。「如何使用」的判据：我们支持的指向本 org；官方已有可用镜像的指向官方；两者皆无的如实说明原因与是否列入计划。版本与状态为 2026-09 调查值，判据与证据链见 [buildkit 的文档](https://github.com/distrotwin/buildkit)。

| OS | 最新版本（发布时间） | 现状 | 如何使用 |
|---|---|---|---|
| 银河麒麟桌面 | V11（2025-08） | 在售维护，安可入围主力 | ✅ 本 org：[`kylin`](https://github.com/distrotwin/kylin) — V4 / V10 / V10 SP1 / V11，V11 含 LoongArch 新世界 |
| 统信 UOS 桌面 | V25（2026-04） | 在售维护，安可入围主力 | ✅ 本 org：[`uos`](https://github.com/distrotwin/uos) — V20 / V25（从官方 ISO 切片），V25 含 LoongArch 新世界 |
| 麒麟信安桌面 | V6 SP1（2026-06） | 在售维护（电力行业） | ✅ 本 org：[`kylinsec`](https://github.com/distrotwin/kylinsec) — V6（含 LoongArch 新世界）/ V3.4 |
| 凝思安全操作系统 | 多支并行，最新公开构建 2026-06 | 在售维护（电力调度），V6.0 伞号下多底座并行 | ✅ 本 org：[`linx`](https://github.com/distrotwin/linx) — 六支全谱系（磐石 4.2 / squeeze / an7 / an8 / buster 主线 / el24.03），glibc 2.5→2.38 |
| 方德桌面 | V5.0（2022-06；Pro 版 2025-09 过安可） | 在售维护，安可入围 | ✅ 本 org：[`fangde`](https://github.com/distrotwin/fangde) — 公开源三代（v3.1 / panda / tiger） |
| Loongnix 桌面 | 25.1（2026-08） | 社区活跃（龙芯官方社区版） | ✅ 本 org：[`loongnix`](https://github.com/distrotwin/loongnix) — 25 线 loong64（glibc 2.41）；20 线为旧世界 ABI，上游 QEMU 造不出（判据见 buildkit 文档） |
| openKylin | 3.0（2026-08） | 社区活跃 | ✔ 官方已有：`docker pull openkylin/openkylin:3.0`（实测可匿名拉取），不重复建设 |
| deepin | 25.2.1（2026-08） | 社区活跃 | ✔ 官方已有：`docker pull linuxdeepin/deepin`（实测可匿名拉取），不重复建设 |
| openEuler | 24.03 LTS SP4（2026-06） | 社区活跃（严格说是服务器系，桌面以软件包提供） | ✔ 官方已有：`docker pull openeuler/openeuler:24.03-lts`，另有 DevStation 官方 rootfs tar |
| Ubuntu Kylin（优麒麟） | 26.04.1 LTS（2026-08） | 社区活跃 | 官方无容器镜像。**不列入计划**：底座是 Ubuntu 直系，构建测试用官方 `ubuntu` 镜像即基本同质，自建无 ABI 增量 |
| Anolis OS（龙蜥） | 23.5 | 社区活跃（服务器定位） | ✔ 官方已有 `openanolis/anolisos`（服务器镜像）。桌面无独立产品线（DDE 仅为附属仓库），桌面向**不列入计划** |
| OpenCloudOS | 9.6（2026-07） | 社区活跃（服务器定位） | ✔ 官方已有 `opencloudos/opencloudos:9.0`。桌面无独立 ISO，桌面向**不列入计划** |
| AOSC OS（安同） | 滚动（2026-06 快照） | 社区活跃 | 官方发布 rootfs tarball（可直接 `docker import`），无 registry 镜像。**不列入计划**：滚动发行 + 自研包管理 oma，与本 org「钉住厂商发布快照」的形态不合，且官方 tarball 已可直接使用 |
| RevyOS | 20260504 快照 | 社区活跃（RISC-V） | 官方为刷机镜像。**不列入计划**：面向 RISC-V 开发板的系统，与桌面软件构建测试的需求源错位；如需接入，QEMU riscv64 技术上可行 |
| Bianbu OS | v4.0.6（2026-08） | 在售（RISC-V，进迭时空） | 官方为开发板镜像。**不列入计划**：同上，RISC-V 开发板向 |
| 新支点桌面（NSDL） | V4.5.2（官方未标日期） | 在售（中兴系） | 无官方镜像。**待定**：官网未见公开 ISO/源直链，材料可得性尚未实测；探明有公开材料即可按现有机制接入 |
| 普华桌面 | 官方两处版本自相矛盾（V4.0 / V5.0） | 状态存疑 | 无官方镜像。**不列入计划**：官方版本信息不一致且无公开介质可验 |
| 中标麒麟桌面（NeoKylin） | V7（历史产品） | 品牌已并入麒麟软件 | **不列入计划**：产品线由银河麒麟延续，已被 [`kylin`](https://github.com/distrotwin/kylin) 覆盖 |
| 一铭桌面 | 4.0 SP1（2016-06） | 停滞（2016 后无公开更新） | 无官方镜像。**不列入计划**：材料不可得 |
| FydeOS | v23 | 在售 | **不列入计划**：ChromiumOS 用户态，常规 Linux 容器范式不适用 |
| EulerOS（华为） | 未公开 | 商业在售（服务器） | **不列入计划**：无桌面产品线，且无公开下载 |

## 本 org 的仓库

| 仓库 | 作用 |
|---|---|
| [`buildkit`](https://github.com/distrotwin/buildkit) | 公共机器码：五条构建路径（mmdebstrap / selfhost / slice / rpmmedia / pkgslice）、数据镜像取材机制、测试与验收门禁、可复用 CI workflow；全部踩坑记录都在它的 `docs/` 里 |
| [`kylin`](https://github.com/distrotwin/kylin) | 银河麒麟 V4 / V10 / V10 SP1 / V11 → [镜像](https://github.com/distrotwin/kylin/pkgs/container/kylin) |
| [`uos`](https://github.com/distrotwin/uos) | 统信 UOS V20 / V25 → [镜像](https://github.com/distrotwin/uos/pkgs/container/uos) |
| [`kylinsec`](https://github.com/distrotwin/kylinsec) | 麒麟信安 V6 / V3.4 → [镜像](https://github.com/distrotwin/kylinsec/pkgs/container/kylinsec) |
| [`loongnix`](https://github.com/distrotwin/loongnix) | Loongnix 桌面 25（loong64） → [镜像](https://github.com/distrotwin/loongnix/pkgs/container/loongnix) |
| [`linx`](https://github.com/distrotwin/linx) | 凝思六支全谱系（glibc 2.5→2.38） → [镜像](https://github.com/distrotwin/linx/pkgs/container/linx) |
| [`fangde`](https://github.com/distrotwin/fangde) | 方德桌面三代（v3.1 / panda / tiger） → [镜像](https://github.com/distrotwin/fangde/pkgs/container/fangde) |
| `scratch` | 数据镜像：厂商站点对 CI 不可达时的介质中转（一个介质一个 tag，带完整性锚点），机制见 buildkit `docs/srcdata.md` |

每个镜像三档（`micro` 最小根系统 / `base` 常用工具 / `devel` 编译工具链），tag 规则、基线数字、该系统的已知怪癖，见各仓库 README。
