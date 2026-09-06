# distrotwin · 国产桌面操作系统的构建与测试镜像

把国产桌面操作系统做成可用于**软件构建、打包与兼容性测试**的容器镜像，公开在 GHCR，匿名可拉。

## 为什么做这个

给国产桌面 OS 交付软件的人都会遇到同一个问题：CI 里没有对应的环境。我们实测了公开名录里 42 个候选容器镜像引用，又直接枚举了麒麟软件与统信自有 registry 的全部公开项目——**主流商业国产桌面 OS 没有一家提供官方的桌面线容器镜像**（麒麟 `cr.kylinos.cn` 全部公开仓库[^1][^2]、统信容器镜像平台全部公开项目[^3][^4]，含 desktop/ukui/dde 字样的均为零）。缺了它，编译要么在真机上手工做，要么拿一个血统相近的社区镜像凑合：前者进不了 CI，后者的 glibc/libstdc++ 基线跟真实目标对不上，验出来的结论不可信。

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

一行一个系统，版本与状态截至 2026-09，每行证据见文末引用（官方页面、社区页面、安可测评公告与镜像仓库，均为原始出处）。现状：🟢 维护中 · 🟡 状态存疑 · ⚫ 停滞或已并入。如何使用：✅ 用本 org 的镜像 · 📦 官方已有镜像（均经匿名拉取实测）· ⏳ 待定 · 🚫 不做（附原因）。

| OS | 最新版本（发布时间） | 现状 | 如何使用 |
|---|---|---|---|
| [银河麒麟桌面](https://www.kylinos.cn/) | V11（2025-08） | 🟢 在售维护，安可公告入围[^5][^6][^7] | ✅ [`kylin`](https://github.com/distrotwin/kylin)：V4 / V10 / V10 SP1 / V11，V11 含 LoongArch |
| [统信 UOS 桌面](https://www.uniontech.com/) | V25（2026-04） | 🟢 在售维护，安可公告入围[^8][^9][^10] | ✅ [`uos`](https://github.com/distrotwin/uos)：V20 / V25（官方 ISO 切片），V25 含 LoongArch |
| [麒麟信安桌面](https://www.kylinsec.com.cn/) | V6 SP1（2026-06） | 🟢 在售维护（电力行业）[^11][^12] | ✅ [`kylinsec`](https://github.com/distrotwin/kylinsec)：V6（含 LoongArch）/ V3.4 |
| [凝思安全操作系统](https://www.linx-info.com/) | 多支并行，最新构建 2026-06 | 🟢 在售维护（电力调度）[^13] | ✅ [`linx`](https://github.com/distrotwin/linx)：六支全谱系，glibc 2.5→2.38 |
| [方德桌面](https://www.nfschina.com/) | V5.0（2022-06；Pro 版 2025-09 过安可） | 🟢 在售维护，安可公告入围[^14][^15][^7] | ✅ [`fangde`](https://github.com/distrotwin/fangde)：公开源三代（v3.1 / panda / tiger） |
| [Loongnix 桌面](https://www.loongnix.cn/) | 25.1（2026-08） | 🟢 社区活跃（龙芯官方社区版）[^16] | ✅ [`loongnix`](https://github.com/distrotwin/loongnix)：25 线 loong64（glibc 2.41）；20 线是 LoongArch 旧世界 ABI，通用模拟器跑不起来，未收录 |
| [openKylin](https://www.openkylin.top/) | 3.0（2026-08） | 🟢 社区活跃[^17] | 📦 `docker pull openkylin/openkylin:3.0`，官方已覆盖[^18] |
| [deepin](https://www.deepin.org/) | 25.2.1（2026-08） | 🟢 社区活跃[^19] | 📦 `docker pull linuxdeepin/deepin`，官方已覆盖[^20] |
| [openEuler](https://www.openeuler.org/) | 24.03 LTS SP4（2026-06） | 🟢 社区活跃（服务器系，桌面以软件包提供）[^21][^22] | 📦 `docker pull openeuler/openeuler:24.03-lts`[^23]，另有 DevStation 官方 rootfs tar[^24] |
| [Ubuntu Kylin（优麒麟）](https://www.ubuntukylin.com/) | 26.04.1 LTS（2026-08） | 🟢 社区活跃[^25][^26] | 🚫 不做：底座是 Ubuntu 直系，官方 `ubuntu` 镜像即基本同质，自建没有 ABI 增量 |
| [Anolis OS（龙蜥）](https://openanolis.cn/anolisos/23) | 23.5 | 🟢 社区活跃（服务器定位）[^27] | 📦 服务器镜像官方已有（`openanolis/anolisos`）[^28]；桌面无独立产品线（DDE 为附属仓库[^29]），桌面向不做 |
| [OpenCloudOS](https://www.opencloudos.org/) | 9.6（2026-07） | 🟢 社区活跃（服务器定位）[^30] | 📦 服务器镜像官方已有（`opencloudos/opencloudos:9.0`）[^31]；桌面无独立 ISO（官方文档为包安装方式[^32]），桌面向不做 |
| [AOSC OS（安同）](https://aosc.io/) | 滚动（2026-06 快照） | 🟢 社区活跃[^33] | 🚫 不做：滚动发行加自研包管理 oma，不适合做成固定版本快照；官方 rootfs tarball 可直接 `docker import` 使用[^33] |
| [RevyOS](https://docs.revyos.dev/) | 20260504 快照 | 🟢 社区活跃（RISC-V）[^34][^35] | 🚫 不做：定位是 RISC-V 开发板系统，不在桌面构建测试范围 |
| [Bianbu OS](https://bianbu.spacemit.com/) | v4.0.6（2026-08） | 🟢 在售（RISC-V，进迭时空）[^36][^37] | 🚫 不做：同上，RISC-V 开发板向 |
| [新支点桌面（NSDL）](https://www.gd-linux.com/) | V4.5.2（官方未标日期） | 🟢 在售（中兴系）[^38] | ⏳ 待定：官网未见公开 ISO/源直链，材料可得性尚未实测；探明有公开材料即可接入 |
| [普华桌面](https://www.i-soft.com.cn/) | 官方两处版本不一致（V4.0 / V5.0） | 🟡 状态存疑[^39][^40] | 🚫 不做：官方版本信息不一致，且无公开介质可验 |
| [中标麒麟桌面（NeoKylin）](https://www.cs2c.com.cn/) | V7（历史产品） | ⚫ 品牌已并入麒麟软件[^41][^42] | 🚫 不做：产品线由银河麒麟延续，已被 [`kylin`](https://github.com/distrotwin/kylin) 覆盖 |
| [一铭桌面](http://www.emindsoft.com.cn/index.php) | 4.0 SP1（2016-06） | ⚫ 停滞（2016 后无公开更新）[^43][^44] | 🚫 不做：材料不可得 |
| [FydeOS](https://fydeos.com/) | v23 | 🟢 在售[^45] | 🚫 不做：ChromiumOS 用户态，常规 Linux 容器方式装不出等价环境 |
| [EulerOS（华为）](https://www.huaweicloud.com/product/euleros.html) | 未公开 | 🟢 商业在售（服务器）[^46][^47] | 🚫 不做：无桌面产品线，也无公开下载[^48] |

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

## 引用

[^1]: 麒麟软件容器仓库 公开项目列表（JSON API）· cr.kylinos.cn · <https://cr.kylinos.cn/api/v2.0/projects>（访问 2026-08-30）
[^2]: 麒麟软件容器仓库 kylin 项目全部 27 个仓库（JSON API，含 desktop/ukui 者为零）· cr.kylinos.cn · <https://cr.kylinos.cn/api/v2.0/projects/kylin/repositories>（访问 2026-08-30）
[^3]: 统信容器镜像平台 公开项目列表（JSON API）· registry.uniontech.com · <https://registry.uniontech.com/api/v2.0/projects>（访问 2026-08-30）
[^4]: 统信容器镜像平台 门户页 · uniontech.com · <https://www.uniontech.com/os-containerImage.html>（访问 2026-08-30）
[^5]: 银河麒麟桌面操作系统 V11 产品页 · kylinos.cn · <https://www.kylinos.cn/productPc/desktop/desktopMainV11/>（访问 2026-08-30）
[^6]: 首个商用 6.6 内核中国操作系统银河麒麟 V11 发布 · 科技日报 · <https://www.stdaily.com/web/gdxw/2025-08/26/content_390577.html>（访问 2026-08-30）
[^7]: 安全可靠测评结果公告（2025 年第 3 号）：银河麒麟桌面 V11、方德桌面 Pro 版 V5.0 入围 · itsec.gov.cn · <https://www.itsec.gov.cn/aqkkcp/cpgg/202509/t20250912_235645.html>（访问 2026-08-30）
[^8]: 统信桌面操作系统 V25 发布 · uniontech.com · <https://uniontech.com/m/news-info/2877.html>（访问 2026-08-30）
[^9]: 统信桌面操作系统 V25 专业版 产品页 · uniontech.com · <https://www.uniontech.com/OS-Professional-edition-V25.html>（访问 2026-08-30）
[^10]: 安全可靠测评结果公告（2026 年第 1 号）：统信桌面 V25 入围 · itsec.gov.cn · <https://www.itsec.gov.cn/aqkkcp/cpgg/202601/t20260116_249958.html>（访问 2026-08-30）
[^11]: 麒麟信安操作系统 桌面版 产品页 · kylinsec.com.cn · <https://www.kylinsec.com.cn/detail/1413/1.html>（访问 2026-08-30）
[^12]: 麒麟信安 ISO 镜像列表（匿名 autoindex，9 个版本目录）· mirrorlists.kylinsec.com.cn · <https://mirrorlists.kylinsec.com.cn/iso/>（访问 2026-08-30）
[^13]: 凝思官方下载目录（匿名可列，多支版本并行、ISO 匿名可下）· linx-info.com · <https://www.linx-info.com/download/os/>（访问 2026-09-06）
[^14]: 中科方德官网 · nfschina.com · <https://www.nfschina.com/>（访问 2026-09-06）
[^15]: 安全可靠测评结果公告（2024 年第 1 号）：方德桌面 V5.0 入围 · itsec.gov.cn · <https://www.itsec.gov.cn/aqkkcp/cpgg/202405/t20240520_172866.html>（访问 2026-08-30）
[^16]: Loongnix 龙芯开源社区官网（25.1 / 20.7 版本入口）· loongnix.cn · <https://www.loongnix.cn/>（访问 2026-09-06）
[^17]: openKylin 3.0 正式发布 · openkylin.top · <https://www.openkylin.top/news/4099-en.html>（访问 2026-08-30）
[^18]: Docker Hub · openkylin/openkylin（`:3.0` 于 2026-09-06 经 `docker manifest inspect` 匿名实测可拉）· <https://hub.docker.com/r/openkylin/openkylin>
[^19]: deepin 25.2 Release Note · deepin.org · <https://www.deepin.org/en/deepin-25-2-release/>（访问 2026-08-30）
[^20]: Docker Hub · linuxdeepin/deepin（2026-09-06 匿名实测可拉）· <https://hub.docker.com/r/linuxdeepin/deepin>
[^21]: openEuler 下载页（24.03 LTS SP4）· openeuler.org · <https://www.openeuler.org/zh/download/>（访问 2026-08-30）
[^22]: openEuler 文档 · 图形桌面使用（桌面以软件包提供）· docs.openeuler.org · <https://docs.openeuler.org/zh/docs/24.03_LTS_SP4/tools/desktop/index.html>（访问 2026-08-30）
[^23]: Docker Hub · openeuler/openeuler（`:24.03-lts` 于 2026-09-06 匿名实测可拉）· <https://hub.docker.com/r/openeuler/openeuler>
[^24]: openEuler DevStation 官方容器 rootfs（tar.xz，匿名可下）· repo.openeuler.org · <https://repo.openeuler.org/openEuler-24.03-LTS-SP3/DevStation/x86_64/docker_img/openEuler-docker.x86_64.tar.xz>（访问 2026-08-30）
[^25]: 优麒麟 26.04 版本正式发布 · ubuntukylin.com · <https://www.ubuntukylin.com/news/ubuntukylin2604-cn.html>（访问 2026-08-30）
[^26]: Ubuntu Kylin 26.04.1 LTS 官方 ISO 目录 · cdimage.ubuntu.com · <https://cdimage.ubuntu.com/ubuntukylin/releases/26.04.1/release/>（访问 2026-08-30）
[^27]: Anolis OS 23 产品页（内核 6.6 LTS）· openanolis.cn · <https://openanolis.cn/anolisos/23>（访问 2026-08-30）
[^28]: Docker Hub · openanolis/anolisos（2026-09-06 匿名实测可拉）· <https://hub.docker.com/r/openanolis/anolisos>
[^29]: Anolis 23.5 DDE 附属仓库目录 · mirrors.openanolis.cn · <https://mirrors.openanolis.cn/anolis/23.5/DDE/>（访问 2026-08-30）
[^30]: OpenCloudOS v9.6 发行说明 · docs.opencloudos.org · <https://docs.opencloudos.org/release/v9.6/>（访问 2026-08-30）
[^31]: Docker Hub · opencloudos/opencloudos（`:9.0` 于 2026-09-06 匿名实测可拉；`:latest` 不存在）· <https://hub.docker.com/r/opencloudos/opencloudos>
[^32]: OpenCloudOS 文档 · 桌面安装（包安装方式，无独立桌面 ISO）· docs.opencloudos.org · <https://docs.opencloudos.org/OCS/Install_Guide/ocs-desktop/>（访问 2026-08-30）
[^33]: 安同开源社区 下载中心（滚动快照与 rootfs tarball）· aosc.io · <https://aosc.io/downloads/>（访问 2026-09-06）
[^34]: RevyOS 官方文档站 · docs.revyos.dev · <https://docs.revyos.dev/>（访问 2026-09-06）
[^35]: RevyOS 镜像目录（20260504 快照）· mirror.iscas.ac.cn · <https://mirror.iscas.ac.cn/revyos/>（访问 2026-08-30）
[^36]: Bianbu OS 官网 · bianbu.spacemit.com · <https://bianbu.spacemit.com/>（访问 2026-09-06）
[^37]: Bianbu K1 镜像版本目录（v4.0.6）· archive.spacemit.com · <https://archive.spacemit.com/image/k1/version/>（访问 2026-08-31）
[^38]: 中兴新支点 公司与产品介绍页 · gd-linux.com · <https://www.gd-linux.com/about/>（访问 2026-09-06）
[^39]: 普华桌面操作系统 V4.0 产品详情页 · i-soft.com.cn · <https://www.i-soft.com.cn/detail/59.html>（访问 2026-08-30）
[^40]: 普华通用产品页 存档（与产品详情页版本表述不一致处）· web.archive.org · <http://web.archive.org/web/20260608023757/https://www.i-soft.com.cn/product/currency.html>（抓取 2026-06-08）
[^41]: 中标软件官网（中标麒麟原厂）· cs2c.com.cn · <https://www.cs2c.com.cn/>（访问 2026-09-06）
[^42]: 麒麟软件 公司介绍（中标软件与天津麒麟整合沿革）· kylinos.cn · <https://www.kylinos.cn/about/company/index.html>（访问 2026-09-06）
[^43]: 一铭软件官网 · emindsoft.com.cn · <http://www.emindsoft.com.cn/index.php>（访问 2026-08-30；2026-09-06 直连已无响应）
[^44]: 一铭桌面操作系统 4.0 SP1 页面存档（2016-06）· web.archive.org · <http://web.archive.org/web/20230401004048/http://emindsoft.com.cn/>（抓取 2023-04-01）
[^45]: FydeOS v23 发布说明 · fydeos.com · <https://fydeos.com/blog/release-note-v23/>（访问 2026-08-30）
[^46]: 华为云 EulerOS 产品页 · huaweicloud.com · <https://www.huaweicloud.com/product/euleros.html>（访问 2026-09-06）
[^47]: openEuler 商业发行版列表（含 EulerOS）· openeuler.org · <https://www.openeuler.org/zh/download/commercial-release/>（访问 2026-08-30）
[^48]: 华为镜像站 EulerOS 目录（仅 RPM 仓库，无 ISO 公开下载）· repo.huaweicloud.com · <https://repo.huaweicloud.com/euleros/>（访问 2026-08-30）
