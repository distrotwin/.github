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
| [银河麒麟桌面](https://www.kylinos.cn/) | V11（2025-08） | 🟢 在售维护，安可入围[^5][^6][^7]；中央国家机关台式/便携机批量集采将其列为预装标配操作系统[^49]，并入围央采 Linux 协议供货（采购单位覆盖中央直属近万个机关单位）[^50] | ✅ [`kylin`](https://github.com/distrotwin/kylin)：V4 / V10 / V10 SP1 / V11，V11 含 LoongArch |
| [统信 UOS 桌面](https://www.uniontech.com/) | V25（2026-04） | 🟢 在售维护，安可入围[^8][^9][^10]；央采批量集采预装标配之一[^49]，2026 年央采台式机中标机型即深度适配其系统[^51] | ✅ [`uos`](https://github.com/distrotwin/uos)：V20 / V25（官方 ISO 切片），V25 含 LoongArch |
| [麒麟信安桌面](https://www.kylinsec.com.cn/) | V6 SP1（2026-06） | 🟢 在售维护[^11][^12]；入围中央国家机关 2024 年度操作系统框架协议采购[^52]，2025 半年报披露中标国网/南网集采及浙江、沈阳等多地新一代调度系统云桌面项目[^53] | ✅ [`kylinsec`](https://github.com/distrotwin/kylinsec)：V6（含 LoongArch）/ V3.4 |
| [凝思安全操作系统](https://www.linx-info.com/) | 多支并行，最新构建 2026-06 | 🟢 在售维护[^13]；深交所披露文件显示其收入 96% 以上来自电力行业、国家电网单一客户占四成以上，产品用于国网 D5000 调度系统等核心业务[^54][^55] | ✅ [`linx`](https://github.com/distrotwin/linx)：六支全谱系，glibc 2.5→2.38 |
| [方德桌面](https://www.nfschina.com/) | V5.0（2022-06；Pro 版 2025-09 过安可） | 🟢 在售维护，安可入围[^14][^15][^7]；央采批量集采预装标配之一[^49] | ✅ [`fangde`](https://github.com/distrotwin/fangde)：公开源三代（v3.1 / panda / tiger） |
| [Loongnix 桌面](https://www.loongnix.cn/) | 25.1（2026-08） | 🟢 社区活跃[^16]；官方定位为龙芯平台商业版/定制版操作系统的技术源头，主要使用者为龙芯整机厂商、云厂商与开发者[^56] | ✅ [`loongnix`](https://github.com/distrotwin/loongnix)：25 线 loong64（glibc 2.41）；20 线是 LoongArch 旧世界 ABI，通用模拟器跑不起来，未收录 |
| [openKylin](https://www.openkylin.top/) | 3.0（2026-08） | 🟢 社区活跃[^17]；面向个人桌面与开发者的开源根社区，政企端由商业版银河麒麟承接[^57] | 📦 `docker pull openkylin/openkylin:3.0`，官方已覆盖[^18] |
| [deepin](https://www.deepin.org/) | 25.2.1（2026-08） | 🟢 社区活跃；社区用户超 760 万（统信官方口径），政企端由其商业版统信 UOS 承接[^19][^58] | 📦 `docker pull linuxdeepin/deepin`，官方已覆盖[^20] |
| [openEuler](https://www.openeuler.org/) | 24.03 LTS SP4（2026-06） | 🟢 社区活跃（服务器系，桌面以软件包提供）[^21][^22]；开源五年累计装机超 1000 万套，规模落地于运营商、金融等行业[^59] | 📦 `docker pull openeuler/openeuler:24.03-lts`[^23]，另有 DevStation 官方 rootfs tar[^24] |
| [Ubuntu Kylin（优麒麟）](https://www.ubuntukylin.com/) | 26.04.1 LTS（2026-08） | 🟢 社区活跃[^25][^26]；官方社区口径全球下载超 3800 万次、开发者与爱好者 20 万+，主要面向个人用户[^60] | 🚫 不做：底座是 Ubuntu 直系，官方 `ubuntu` 镜像即基本同质，自建没有 ABI 增量 |
| [Anolis OS（龙蜥）](https://openanolis.cn/anolisos/23) | 23.5 | 🟢 社区活跃（服务器定位）[^27]；装机量超 800 万（阿里云官方口径），主要用于 CentOS 替代的服务器场景[^61] | 📦 服务器镜像官方已有（`openanolis/anolisos`）[^28]；桌面无独立产品线（DDE 为附属仓库[^29]），桌面向不做 |
| [OpenCloudOS](https://www.opencloudos.org/) | 9.6（2026-07） | 🟢 社区活跃（服务器定位）[^30]；装机超 1500 万节点，落地于金融、政务等服务器场景（腾讯主导社区口径）[^62] | 📦 服务器镜像官方已有（`opencloudos/opencloudos:9.0`）[^31]；桌面无独立 ISO（官方文档为包安装方式[^32]），桌面向不做 |
| [AOSC OS（安同）](https://aosc.io/) | 滚动（2026-06 快照） | 🟢 社区活跃[^33]；志愿者社区维护，面向有 Linux 经验的个人桌面用户，无政企采购形态[^63] | 🚫 不做：滚动发行加自研包管理 oma，不适合做成固定版本快照；官方 rootfs tarball 可直接 `docker import` 使用[^33] |
| [RevyOS](https://docs.revyos.dev/) | 20260504 快照 | 🟢 社区活跃（RISC-V）[^34][^35]；中科院软件所 PLCT/RuyiSDK 团队维护，主要使用者为玄铁 RISC-V 芯片开发者[^64] | 🚫 不做：定位是 RISC-V 开发板系统，不在桌面构建测试范围 |
| [Bianbu OS](https://bianbu.spacemit.com/) | v4.0.6（2026-08） | 🟢 在售（RISC-V，进迭时空）[^36][^37]；作为进迭时空 K1 开发板与整机的默认系统随硬件出货，主要使用者为 RISC-V 开发者[^65] | 🚫 不做：同上，RISC-V 开发板向 |
| [新支点桌面（NSDL）](https://www.gd-linux.com/) | V4.5.2（官方未标日期） | 🟢 在售（中兴系）[^38]；产品在架中央政采电子卖场[^66]，2026 年央采服务器操作系统框架协议入围[^67]，2024 年入围深圳政采框架协议[^68] | ⏳ 待定：官网未见公开 ISO/源直链，材料可得性尚未实测；探明有公开材料即可接入 |
| [普华桌面](https://www.i-soft.com.cn/) | 官方两处版本不一致（V4.0 / V5.0） | 🟡 状态存疑[^39][^40]；行业调研称其历史上曾在近 60 个部委推广应用[^69]，近年未检索到公开中标公告 | 🚫 不做：官方版本信息不一致，且无公开介质可验 |
| [中标麒麟桌面（NeoKylin）](https://www.cs2c.com.cn/) | V7（历史产品） | ⚫ 品牌已并入麒麟软件[^41][^42]；央采协议供货现由麒麟软件（旗下含银河麒麟与中标麒麟两品牌）延续入围[^50] | 🚫 不做：产品线由银河麒麟延续，已被 [`kylin`](https://github.com/distrotwin/kylin) 覆盖 |
| [一铭桌面](http://www.emindsoft.com.cn/index.php) | 4.0 SP1（2016-06） | ⚫ 停滞（2016 后无公开更新）[^43][^44]；未检索到可核实的公开采购或使用记录 | 🚫 不做：材料不可得 |
| [FydeOS](https://fydeos.com/) | v23 | 🟢 在售[^45]；商业形态是面向企业与教育行业的定制部署与订阅服务（官方发布行业案例集）[^70] | 🚫 不做：ChromiumOS 用户态，常规 Linux 容器方式装不出等价环境 |
| [EulerOS（华为）](https://www.huaweicloud.com/product/euleros.html) | 未公开 | 🟢 商业在售（服务器）[^46][^47]；华为自用商用系统，搭载于其通信设备与华为云自有业务[^71] | 🚫 不做：无桌面产品线，也无公开下载[^48] |

## 本 org 的仓库

| 仓库 | 作用 | 架构 |
|---|---|---|
| [`buildkit`](https://github.com/distrotwin/buildkit) | 公共构建机器码：五条构建路径、数据镜像取材、测试与验收门禁、可复用 CI workflow；机制文档与已知问题记录在 `docs/` | — |
| [`kylin`](https://github.com/distrotwin/kylin) | 银河麒麟 V4 / V10 / V10 SP1 / V11 → [镜像](https://github.com/distrotwin/kylin/pkgs/container/kylin) | 全版本 amd64 + arm64；**V11 另有 loong64** |
| [`uos`](https://github.com/distrotwin/uos) | 统信 UOS V20 / V25 → [镜像](https://github.com/distrotwin/uos/pkgs/container/uos) | 全版本 amd64 + arm64；**V25 另有 loong64** |
| [`kylinsec`](https://github.com/distrotwin/kylinsec) | 麒麟信安 V6 / V3.4 → [镜像](https://github.com/distrotwin/kylinsec/pkgs/container/kylinsec) | 全版本 amd64 + arm64；**V6 另有 loong64** |
| [`loongnix`](https://github.com/distrotwin/loongnix) | Loongnix 桌面 25 → [镜像](https://github.com/distrotwin/loongnix/pkgs/container/loongnix) | **loong64**（纯龙芯仓库） |
| [`linx`](https://github.com/distrotwin/linx) | 凝思六支全谱系（glibc 2.5→2.38） → [镜像](https://github.com/distrotwin/linx/pkgs/container/linx) | 全支 amd64；6.0.98 / 6.0.99 另有 arm64 |
| [`fangde`](https://github.com/distrotwin/fangde) | 方德桌面三代（v3.1 / panda / tiger） → [镜像](https://github.com/distrotwin/fangde/pkgs/container/fangde) | v3.1 为 amd64；panda / tiger 为 amd64 + arm64 |
| `scratch` | 数据镜像：厂商站点对 CI 不可达时的介质中转，一个介质一个 tag，带完整性锚点 | 随介质 |

每个镜像三档：`micro`（最小根系统）/ `base`（常用工具）/ `devel`（编译工具链）。tag 规则、基线数字与各系统的已知怪癖见对应仓库的 README。

**关于 LoongArch 要单独说两句。** 龙芯有新旧两套互不兼容的 ABI：新世界（`loong64`，动态链接器 `ld-linux-loongarch-lp64d.so.1`）我们已支持三家四线——银河麒麟 V11、统信 V25、麒麟信安 V6、Loongnix 25，这大概是目前唯一能在公共 CI 里直接拉到的国产桌面 OS loong64 构建环境。旧世界（`loongarch64`，链接器 `/lib64/ld.so.1`，银河麒麟 V10 SP1、Loongnix 20、方德的三处龙芯树都属此类）**目前不支持**：上游 QEMU 未实现旧世界的信号系统调用，托管 CI 的模拟环境里装不了包，排查记录见 [buildkit 文档](https://github.com/distrotwin/buildkit)。要留意 rpm 生态里两个世界都叫 `loongarch64`，架构名不携带世代信息——判据在动态链接器上。

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
[^49]: 关于更新中央国家机关台式计算机、便携式计算机批量集中采购配置标准的通知（操作系统要求预装麒麟/统信UOS/中科方德或其他符合安全可靠测评要求的操作系统）· zycg.gov.cn · <https://zycg.gov.cn/freecms/site/zygjjgzfcgzx/tzgg/info/2024/8333657.html>（访问 2026-09-06）
[^50]: 麒麟软件入围中央政府采购协议供货项目名录（中央国家机关 2020-2021 年 Linux 操作系统协议供货，采购单位覆盖中央直属近万个机关单位）· ithome.com · <https://www.ithome.com/0/530/208.htm>（访问 2026-09-06）
[^51]: 紫光计算机连续中标中央国家机关 2026 年台式计算机批量集中采购项目（中标机型深度适配麒麟、统信UOS 等国产操作系统）· unispc.com · <https://www.unispc.com/aboutus/newsfeed/news/1647.html>（访问 2026-09-06）
[^52]: 麒麟信安入围中央国家机关 2024 年度操作系统软件框架协议联合征集采购项目 · kylinsec.com.cn · <https://www.kylinsec.com.cn/detail/21901/1.html>（访问 2026-09-06）
[^53]: 湖南麒麟信安科技股份有限公司 2025 年半年度报告（披露中标国网/南网集采及浙江、沈阳、朝阳、抚顺、营口、大连等地新一代调度系统云桌面项目）· cninfo.com.cn · <http://static.cninfo.com.cn/finalpage/2025-08-29/1224608469.PDF>（访问 2026-09-06）
[^54]: 北京凝思软件股份有限公司创业板 IPO 第二轮审核问询函回复（披露电力行业收入占比 96% 以上、国家电网收入占比 44%-52%、D5000 调度系统部署沿革）· szse.cn · <http://reportdocs.static.szse.cn/UpFiles/rasinfodisc1/202306/RAS_202306_55DDF01C2FB74EF68F48C59FE4115906.pdf>（访问 2026-09-06）
[^55]: 凝思软件冲刺 IPO：客户高度集中在电力系统 · 证券时报 · <https://www.stcn.com/article/detail/935993.html>（访问 2026-09-06）
[^56]: 龙芯中科官网 · Loongnix 操作系统（官方定位：以开源社区版为基础支持商业版和定制版发展）· loongson.cn · <https://www.loongson.cn/system/loongnix>（访问 2026-09-06）
[^57]: 国产操作系统深度测评：个人办公选 openKylin，政企行业选银河麒麟 · C114 通信网 · <https://m.c114.com.cn/w241-1315137.html>（访问 2026-09-06）
[^58]: 统信 UOS AI 2.0 发布，deepin（深度）系统社区用户超 760 万 · 腾讯新闻 · <https://news.qq.com/rain/a/20241211A047SB00>（访问 2026-09-06）
[^59]: openEuler 开源五年 累计装机量突破 1000 万 · 证券时报 · <https://www.stcn.com/article/detail/1414974.html>（访问 2026-09-06）
[^60]: 优麒麟社区介绍（全球下载量 3800+ 万次、活跃爱好者和开发者 20+ 万人）· ubuntukylin.com · <https://ubuntukylin.com/about/ukylin-cn.html>（访问 2026-09-06）
[^61]: 装机量破 800 万台！开源操作系统龙蜥全新发布官方正式版 · 阿里云开发者社区 · <https://developer.aliyun.com/article/1612377>（访问 2026-09-06）
[^62]: 开源操作系统 OpenCloudOS：装机超 1500 万节点 · 腾讯新闻 · <https://news.qq.com/rain/a/20241224A0773000>（访问 2026-09-06）
[^63]: 安同 OS 产品页（社区志愿维护，面向有一定 Linux 使用经验的个人桌面用户）· aosc.io · <https://aosc.io/aosc-os>（访问 2026-09-06）
[^64]: RevyOS 源码仓库（RuyiSDK 旗下，针对玄铁 RISC-V 芯片生态优化的 Debian 定制发行版）· github.com · <https://github.com/ruyisdk/revyos>（访问 2026-09-06）
[^65]: Bianbu Linux 官方文档站（进迭时空 K1 平台默认操作系统）· spacemit.com · <https://bianbu-linux.spacemit.com/>（访问 2026-09-06）
[^66]: 新支点通用操作系统 V4 · 中央政采电子卖场在架商品页 · zycg.gov.cn · <https://mkt.zycg.gov.cn/mall-view/software/detail?skuId=1672800>（访问 2026-09-06）
[^67]: 2026 央采将中兴新支点操作系统列入采购清单（服务器操作系统框架协议入围）· 快科技 · <https://news.mydrivers.com/1/1143/1143543.htm>（访问 2026-09-06）
[^68]: 中兴新支点操作系统被纳入深圳政府采购清单（2024-2025 年度服务器操作系统框架协议采购入围）· 中华网 · <https://m.tech.china.com/tech/article/20241014/102024_1587916.html>（访问 2026-09-06）
[^69]: 企业调研：国产操作系统厂商普华软件（历史上在近 60 个部委获得应用推广）· 安全内参 · <https://www.secrss.com/articles/11535>（访问 2026-09-06）
[^70]: FydeOS 企业解决方案应用案例集（教育、企业办公等行业定制部署）· fydeos.com · <https://resources.fydeos.com/enterprise/Enterprise-Solution-Case-Studies_ZH.pdf>（访问 2026-09-06）
[^71]: 华为云开发者 · EulerOS（华为基于 openEuler 的自用商用操作系统）· developer.huaweicloud.com · <https://developer.huaweicloud.com/euleros/index.html>（访问 2026-09-06）
