# PVE-Manager-Status
为你的 ProxmoxVE 节点概要页面添加扩展的硬件监控信息

无论是小型 HomeLab 还是多节点的大型集群, 借助 PVE-Manager-Status 让你对当前服务器硬件状态了如指掌!

PVE-Manager-Status 是一款强大的开源脚本工具, 通过实时基于动态色彩的关键指标显示, 帮助你直观的了解服务器各项硬件的负载和温度, 从而轻松掌握设备的实时运行状态以确保其稳定高效.

<img width="1120" height="699" alt="image" src="https://github.com/user-attachments/assets/51224701-a763-4b14-9951-5990f7e22901" />

## 免责声明

在任何正式或关键的生产环境中, 未经完整测试与评估, 严禁直接部署和运行本工具. 在使用本工具之前, 务必对代码进行详尽的审阅, 充分理解其运行机制, 使用该工具后的风险由您自行承担.

## 包含的功能

- 为ProxmoxVE Web 管理界面的节点概要中添加各项硬件的实时监控信息.
- 根据各项硬件的温度和负载, 使用红黄绿色从高到低来显示各项参数的数值.
- 支持识别多块 NVMe 硬盘, 并根据寿命和压力负载显示动态色彩指标.
- 支持逐盘读取 SATA/SCSI/SAS 硬盘的 SMART JSON 数据, 显示型号、容量、温度、通电时间和健康状态; 缺少部分指标时仍显示设备.
- 完善 zh-CN 本地化, 修复中文环境下缺失的部分翻译.

## 如何使用

若要安装最新版本, 请通过 ssh 以 root 身份连接你的 Proxmox VE 节点, 并执行以下操作:

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/GCCan/PVE-Manager-Status/main/pve-manager-status.sh)"
```

执行完毕后, 使用 Ctrl + F5 刷新浏览器 Proxmox VE Web 管理页面缓存.

上面的链接指向本 fork 的 `main` 分支. 更新时再次执行同一命令即可; 请先确认修复已推送到 GitHub. 完整脚本会重新安装 `pve-manager` 和 `pve-i18n`, 并重新配置其他监控功能.

若更新后 SATA 数据仍未刷新, 执行 `systemctl restart pvedaemon pveproxy`, 再使用 Ctrl + F5 刷新页面.

若要安装特定版本, 请在项目 Releases 页面选择安装. 

> [!WARNING]
> 若此时可以看到 PVE-Manager-Status 带来的扩展硬件监控信息, 但无法看到风扇转速信息, 这代表 sensors 命令输出中不包含风扇转速信息, 你可能需要安装额外的传感器驱动包. 不同厂商不同设备间使用的接口都可能有所不同, 在此无法一概而论.

对于使用 IT87 系列传感器的设备, 使用以下命令安装传感器驱动包 it87-dkms_1.0.63-1_all.deb

```
wget -O /root/it87-dkms_1.0.63-1_all.deb https://raw.githubusercontent.com/MiKing233/PVE-Manager-Status/master/it87-dkms_1.0.63-1_all.deb && apt install /root/it87-dkms_1.0.63-1_all.deb && rm -f /root/it87-dkms_1.0.63-1_all.deb
```

安装完成后, 重启系统并再次检查风扇转速信息.

## 已知问题

- SATA 修复已通过本地模拟测试和语法检查, 尚未在真实 Proxmox VE 节点和各类控制器上完成硬件验证.
- SATA SMART 扫描和读取可能唤醒休眠硬盘. 采集在状态请求中同步执行, 每次扫描或逐盘查询使用 5 秒超时; 多块慢盘仍可能使概要页面刷新变慢.
- 直通给虚拟机的控制器、不支持 SMART 透传的 USB 桥接设备, 或 smartctl 无法发现的 RAID 物理盘可能无法获取数据. `--scan-open` 不保证发现所有控制器下的全部物理盘, 此时需按控制器型号确认设备路径和 `-d` 参数.
- nvme 硬盘监控部分对于 I/O 实时读写速度和延迟的实现代码使用目前常见的 iostat -d -x -k 1 1 但这将导致长时间运行后 iostat 进程休眠或被杀死, 你将会看到这部分监控参数"冻结", 在重启前将永远不再刷新, 目前尝试过使用其它方式实现这个功能避免冻结, 但都未能达到预期的效果, 有些实现方式复杂超出预期短期内无法解决, 另外一些能生效的方法则不够优雅, 会导致Web页面自动刷新间隔变得非常长.

## SATA 数据采集与排查

本 fork 修复了原命令将多个 `/dev/sd?` 设备一次传给 `smartctl` 的问题, 并改用 JSON 字段解析, 避免依赖不同厂商的 SMART 文本格式.

- 使用 `smartctl --scan-open -j` 发现设备, 保留扫描返回的 `sat`、`scsi`、`megaraid,N` 等设备类型, 对每个设备分别查询.
- 扫描未覆盖的磁盘回退到 `/sys/block/sd*` 整盘列表, 包括 `sdaa` 等名称, 排除分区; NVMe 数据继续使用原来的独立监控项.
- 按设备路径和类型区分 RAID 物理盘. 某块盘缺少温度或通电时间时不会导致整块盘消失, 读取错误会显示在对应条目中.
- 保留原来的温度颜色阈值和旧文本显示兼容逻辑, CPU、传感器及 NVMe 采集逻辑保持不变.

### 检查设备与控制器类型

在 Proxmox VE 节点上以 root 执行:

```bash
smartctl --version
lsblk -d -o NAME,TYPE,TRAN,MODEL
smartctl --scan-open -j
```

使用扫描报告的路径和类型单独查询. 下面的 `sat` 和 `/dev/sda` 仅为示例, 应替换为本机结果:

```bash
smartctl -a -j -d sat /dev/sda
```

如果 root 能读取而页面不能, 再检查 Web 用户权限:

```bash
SMARTCTL_BIN="$(command -v smartctl)"
sudo -u www-data sudo -n "$SMARTCTL_BIN" --scan-open -j
sudo -u www-data sudo -n "$SMARTCTL_BIN" -a -j -d sat /dev/sda
visudo -c -f /etc/sudoers.d/pve-manager-status
```

新命令需要 `--scan-open -j` 和 `-a -j -d TYPE DEVICE` 对应的 sudoers 权限; 只替换 `Nodes.pm` 而保留旧权限规则会导致读取失败. 本仓库完整安装脚本会配置这些权限. `smartctl` 的非零退出码也可能表示 SMART 健康告警, 应结合 JSON 中的 `smart_status` 和诊断信息判断, 不能仅凭退出码认定采集失败.

### 检查状态接口

```bash
perl -c /usr/share/perl5/PVE/API2/Nodes.pm
pvesh get /nodes/"$(hostname -s)"/status --output-format json
```

若节点名与主机短名称不同, 请替换为实际节点名. 返回结果中的 `sata_status` 应为包含逐盘对象的 JSON 数组字符串. 页面应按盘显示独立条目; 多盘、缺少温度或读取失败时不应影响其他磁盘的显示.

本实现依赖支持 `-j` JSON 输出的 smartmontools、Perl `JSON::PP` 和 `/usr/bin/timeout`. PVE 软件包更新可能覆盖定制的页面或 API 文件, 更新后需要重新检查并应用脚本.

## 未来计划

- 修复长时间运行 nvme 硬盘监控部分 I/O读写延迟冻结的问题.

## 相容性

- 适用基于 Debian 12 "bookworm" 的 Proxmox VE 8 与基于 Debian 13 "trixie" 的 Proxmox VE 9.
- 针对 Proxmox VE 7 等旧版环境之可用性未经实际测试, 请谨慎使用本项目. 您需要在拥有root权限的节点上运行该工具执行修改.
- 在将本项目用于实际环境前, 务必先在测试环境进行测试, 若修改后遇到问题请通过下面的命令重装相关软件包以恢复被修改的文件至默认配置.
```
apt install --reinstall pve-manager pve-i18n
```

## 贡献

欢迎您的贡献! 如果您有任何改进或调整, 请 Fork 代码库并提交拉取请求.

## 联系与反馈

如果您遇到任何问题或有任何建议, 请在 GitHub 代码库中提交问题.

## 致谢

感谢以下来源和社区贡献, 本项目的诞生离不开源自这些来源和贡献:

- https://github.com/a1wong/it87
- https://github.com/shidahuilang/pve
- https://github.com/xiangfeidexiaohuo/pve-diy
- https://bbs.x86pi.com/thread?topicId=20
- https://community-scripts.github.io/ProxmoxVE
- https://github.com/community-scripts/ProxmoxVE
- https://github.com/proxmox/pve-manager
- https://github.com/Debian

## 贡献者

<a href="https://github.com/MiKing233/PVE-Manager-Status/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=MiKing233/PVE-Manager-Status" />
</a>

## 许可证

本项目为开源项目, 遵循 MIT License.

这些代码按"原样"提供, 不提供任何担保, 也不授予任何权利. 您有责任安全地使用本工具, 并遵守软件许可协议的条款.

## 请我喝杯咖啡

捐赠是本项目唯一的收入来源, 如果您愿意支持这个项目, 这将帮助该项目走得更远! 感谢您的支持!
- https://donate.mknetwork.net

### 如果你喜欢这个项目, 请不要吝啬您的 Star 🌟
[![Stargazers over time](https://starchart.cc/MiKing233/PVE-Manager-Status.svg?variant=adaptive)](https://starchart.cc/MiKing233/PVE-Manager-Status)

Enjoy a better Proxmox VE experience!
