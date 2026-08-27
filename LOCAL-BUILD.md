# JDCloud RE-CS-02（雅典娜）PPE 固件 — 仓库说明与编译指南

> 本文档面向本地虚拟机编译场景，补充上游 README。CI 用法见 `.github/workflows/build-ppe.yml`。

## 一、本仓库是什么

这是 [huige830/Qualcommax_NSS_Builder](https://github.com/huige830/Qualcommax_NSS_Builder) 的个人 fork，用于给 **京东云 RE-CS-02（雅典娜，IPQ60xx）** 编译带 **PPE 硬件流分载** 的 OpenWrt 固件。

- **上游源码**：`JuliusBairaktaris/openwrt-nss-edma` 的 `ppe-flow-offload` 分支
- **核心特性**：内核 `qca_edma`/`qca_ppe` 数据通路 + netfilter flowtable 硬件卸载、硬件 SQM（`hw_ppe.qos`）、`ppe-qos` 队列分类器（LuCI 界面：Network → PPE QoS）
- **注意**：这是 PPE（host 驱动）路线，**不是 NSS** 路线

## 二、仓库结构

```
builder/
├── devices/
│   ├── common-ppe/config      # PPE 公共配置（所有 PPE 目标共用）
│   ├── ppe-ipq60xx/config      # IPQ60xx 目标配置（RE-CS-02 在此）★ 本机关注
│   └── ...                     # 其他目标（ipq807x 系列、xiaomi_ax3600 等）
├── patches/feeds/ppe/          # feed 补丁（sqm-scripts 重定向到硬件整形 fork）
├── scripts/prepare-build.sh    # CI 的环境准备脚本（feeds 换 luci fork、组装 .config、符号校验）
└── .github/workflows/build-ppe.yml   # CI 流水线
```

## 三、固件内置的第三方包

编译前需克隆进 `openwrt/package/`（CI 和本地脚本都会自动做）：

| 包 | 来源 | 说明 |
|---|---|---|
| OpenClash | vernesong/OpenClash | 代理 |
| luci-app-advanced | sirpdboy | 高级设置 |
| luci-app-partexp | sirpdboy | 分区扩容 |
| luci-app-diskman | sbwml（子目录布局，需提目录） | 磁盘管理 |
| luci-app-easytier | EasyTier | 组网（预编译核心） |
| luci-app-bandix-plus + openwrt-bandix-plus | timsaya | 流量监控（eBPF 预编译后端） |
| athena-led-controller（锁定 v2.4.0） | unraveloop | 雅典娜 LED 点阵屏 |
| luci-app-taskplan | sirpdboy | 定时任务（定时重启/释放内存/清垃圾），autotimeset 后续版 |
| luci-theme-fluent | LazuliKao（子目录布局，需提目录） | Fluent 主题 |

## 四、CI 编译（推荐日常使用）

- **触发**：push 到 main（改 `devices/`、`patches/`、`scripts/`、workflow 本身）自动触发；每日 UTC 3:30 定时检查上游；也可手动 workflow_dispatch
- **产物**：prerelease `ppe-offload-test`，固定链接永久有效，每次构建原地替换
- **只编 ipq60xx**（matrix 已裁剪），固件名为 `openwrt-qualcommax-ipq60xx-jdcloud_re-cs-02-squashfs-sysupgrade.bin`

## 五、本地虚拟机编译（Debian 12，32 线程 / 16G 内存）

### 一次性准备

```bash
# 1. 源码（ppe-flow-offload 分支）
git clone -b ppe-flow-offload https://github.com/JuliusBairaktaris/openwrt-nss-edma.git ~/openwrt
# 2. 本仓库
git clone https://github.com/<你的fork>/Qualcommax_NSS_Builder.git ~/builder
# 3. 编译依赖
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib \
  gettext git libncurses-dev libssl-dev python3-distutils python3-setuptools \
  rsync swig unzip zlib1g-dev
```

### 日常编译（全流程脚本）

```bash
# 完整流程：feeds → 补丁 → .config + 符号校验 → download → 32 线程编译
nohup bash ~/build-re-cs-02.sh > /tmp/build.log 2>&1 &

# 中断后续编（跳过 feeds，复用已编译产物）
bash ~/build-re-cs-02.sh resume
```

脚本位置：`~/build-re-cs-02.sh`（与 CI 的 prepare-build.sh 同款符号校验逻辑，防止静默丢插件；并行失败自动退 `-j1 V=s` 定位）。

### 增量加包（不重跑全流程时）

```bash
cd ~/openwrt
git clone --depth 1 <包仓库> package/<包名>
echo 'CONFIG_PACKAGE_<包名>=y' >> .config
make defconfig
make -j$(nproc)          # 增量，几分钟
```

### 产物与刷机

```
~/openwrt/bin/targets/qualcommax/ipq60xx/
└── openwrt-qualcommax-ipq60xx-jdcloud_re-cs-02-squashfs-sysupgrade.bin
```

```bash
scp .../openwrt-qualcommax-ipq60xx-jdcloud_re-cs-02-squashfs-sysupgrade.bin root@192.168.x.x:/tmp/
ssh root@192.168.x.x sysupgrade -n /tmp/openwrt-*-squashfs-sysupgrade.bin   # -n 不保留配置
```

## 六、踩坑记录（本地环境专属）

1. **PATH 相对路径导致 package/install 失败**（最常见）
   报错：`find: The relative path '...\trae-gopls/current' is included in the PATH environment variable, which is insecure in combination with the -execdir action`
   沙箱/IDE 会往 PATH 注入相对路径，GNU find 拒绝 `-execdir`。解决：编译前清洗 PATH——
   ```bash
   CLEAN_PATH=$(echo "$PATH" | tr ':' '\n' | grep '^/' | paste -sd:)
   PATH="$CLEAN_PATH" nohup make -j$(nproc) > /tmp/build.log 2>&1 &
   ```

2. **OOM（16G 内存 + 32 线程偶发）**
   报错：`cc1: killed` / `internal compiler error`。解决：改 `-j16` 后 `resume` 续编。

3. **下载超时**：直接 `bash ~/build-re-cs-02.sh resume` 重跑即可。

4. **defconfig 符号校验失败**：说明新加的包依赖缺失，把 dropped 清单对照包的 `DEPENDS` 补齐（如 taskplan 需显式 `CONFIG_PACKAGE_bash=y`）。

5. **本地 builder 仓库改了 config 后 `git pull` 冲突**：本地手改的行先 `git stash`，pull 后对比远端是否已含同样改动，重复则 `git stash drop`。

## 七、刷机后验证 PPE 生效

```bash
# 防火墙开启软/硬件流量分载后，查看硬件流表项数
grep -c HW_OFFLOAD /proc/net/nf_conntrack
```

硬件整形：Network → SQM（`hw_ppe.qos`）；队列分类：Network → PPE QoS。
