# OpenWrt ESXi Minimal - 精简旁路由固件

为ESXi虚拟机优化的OpenWrt 24.10精简固件，专用于Photon OS Docker透明代理。

## 功能特性

### 核心功能
- **momo管理界面** - SingBox可视化管理
- **透明代理** - TUN/TPROXY模式，支持Docker流量转发
- **Web SSH** - ttyd终端，浏览器操作命令行
- **精简设计** - 仅30MB固件大小

### 网络配置
- **旁路由模式**: 192.168.1.9（静态IP）
- **主路由**: 192.168.1.8（DHCP服务器）
- **关闭DHCP**: 禁用DHCP/DNS服务，由主路由提供
- **透传防火墙**: 全ACCEPT模式
- **禁用IPv6**: 完全移除IPv6支持

### ESXi优化
- **VMDK格式**: 直接在ESXi使用
- **网卡驱动**: e1000/e1000e/vmxnet3三驱动支持
- **x86-64架构**: ESXi原生支持

## 快速开始

### 使用GitHub Actions构建

1. 点击 `Actions` 标签
2. 选择 `Build OpenWrt 24.10 for ESXi - Minimal` workflow
3. 点击 `Run workflow`
4. 选择或输入OpenWrt版本（默认v24.10.5）
5. 等待构建完成（首次约40-50分钟，缓存后约15-20分钟）

### 下载固件

构建完成后，从Artifacts下载：
- **openwrt-esxi-vmdk**: VMDK格式，直接导入ESXi
- **openwrt-firmware-raw**: 原始.img镜像，本地测试用

### 部署到ESXi

```bash
# 将VMDK文件上传到ESXi数据存储
# 创建新虚拟机，使用现有虚拟磁盘选择openwrt-esxi.vmdk
```

### 本地构建

```bash
# 更新feeds
./scripts/feeds update packages luci

# 配置
make defconfig

# 下载依赖
make download -j8

# 构建
make -j$(nproc)

# 生成VMDK
cd bin/targets/x86/64/
gzip -d *.img.gz
qemu-img convert -f raw -O vmdk -o subformat=monolithicFlat *.img openwrt-esxi-flat.vmdk

# 创建VMDK描述符（手动计算SECTORS）
SIZE=$(stat -c%s openwrt-esxi-flat.vmdk)
SECTORS=$((SIZE / 512))

cat > openwrt-esxi.vmdk <<EOF
# Disk DescriptorFile
version=1
CID=fffffffe
parentCID=ffffffff
createType="monolithicFlat"

# Extent description
RW ${SECTORS} FLAT "openwrt-esxi-flat.vmdk" 0

# The Disk Data Base
#DDB

ddb.adapterType = "lsilogic"
ddb.geometry.cylinders = "522"
ddb.geometry.heads = "255"
ddb.geometry.sectors = "63"
ddb.virtualHWVersion = "14"
EOF
```

## 配置说明

### 首次启动

首次启动会自动执行：
- 设置LuCI中文界面
- 配置网络为192.168.1.9/24，网关192.168.1.8
- 启用ttyd Web SSH（端口7681）
- 禁用IPv6和DHCP
- 配置防火墙透传模式

### 访问Web界面

- **LuCI管理**: http://192.168.1.9
- **Web SSH**: http://192.168.1.9:7681

### SingBox配置

1. 通过LuCI登录
2. 进入 `服务` → `momo`
3. 订阅或手动添加节点
4. momo会自动下载和更新SingBox核心

## 优化记录

### 固件大小优化（200MB → 30MB）

**问题**: GitHub Actions构建约200MB，而openwrt.ai仅30MB

**解决方案**:
- rootfs分区: 256MB → 64MB
- 禁用除中文外的所有语言包
- 移除feeds install -a（避免自动安装额外包）
- 禁用不需要的ipt模块和LuCI协议
- 启用strip选项（CONFIG_USE_SSTRIP=y）

### 构建速度优化（60分钟 → 15-20分钟）

**问题**: GitHub Actions构建需1小时，openwrt.ai仅需5分钟

**解决方案**:
- 添加三层缓存（dl、feeds、build_dir）
- 只更新必要的feeds（packages和luci，而非全部）
- 增强磁盘清理（boost、apt缓存）
- 添加编译优化环境变量

**首次构建**: 40-50分钟  
**缓存命中**: 15-20分钟

### 源地址修正

**问题**: GitHub Actions在海外，应使用国外源

**解决方案**:
- 移除清华镜像配置
- 使用OpenWrt官方源
- feeds保持默认配置

### VMDK转换修复

**问题**: 构建后VMDK转换报错

**解决方案**:
- 添加详细调试输出（文件列表、大小信息）
- 改进镜像查找逻辑（不依赖特定文件名）
- 验证qemu-img可用性和转换结果
- 使用正确的VMDK subformat参数
- 失败时立即退出并显示错误

### 上传流程优化

**问题**: VMDK转换失败需重新构建

**解决方案**:
- 在转换前先上传原始固件
- VMDK转换成功才上传VMDK文件
- 转换失败时可直接下载原始固件测试
- Artifact保留期延长至30天

**上传的文件**:
- `openwrt-firmware-raw`: 原始固件
- `openwrt-esxi-vmdk`: VMDK文件（仅转换成功时）

## 精简策略

### 保留组件
- firewall4（momo依赖）
- 内核模块：TUN/TAP/TPROXY/NFT
- LuCI基础 + momo界面 + ttyd
- 网络工具：curl/wget + ca-certificates
- ESXi驱动：e1000/e1000e/vmxnet3

### 移除组件
- IPv6支持（完全）
- PPP拨号相关（全部）
- DNS/DHCP服务器（用主路由的）
- 无线驱动
- USB支持
- 其他不必要的服务

## 更新策略

- **固件**: 稳定版本，不频繁更新
- **SingBox核心**: 通过momo界面在线更新
- **订阅配置**: 支持自动更新

## 文件结构

```
.
├── config.seed              # OpenWrt配置文件
├── files/
│   ├── 99-custom-network    # 网络配置（旁路由模式）
│   └── 99-firstboot          # 首次启动配置（中文+ttyd）
└── .github/
    └── workflows/
        └── build-openwrt.yml # GitHub Actions构建流程
```

## 故障排查

### VMDK转换失败

1. 从Artifacts下载 `openwrt-firmware-raw`
2. 使用本地qemu-img手动转换
3. 检查日志定位错误原因

### 构建速度慢

- 检查缓存是否命中
- 查看网络连接速度
- 减少构建并行数

### 固件过大

- 检查config.seed中是否禁用所有不需要的包
- 确认rootfs分区大小设置为64MB
- 查看构建日志中是否有额外包被安装

## 许可证

基于OpenWrt和momo项目的开源许可。
