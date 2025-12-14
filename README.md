# EdgeRouter X 针对 ImmortalWrt 24.10 的迁移脚本

EdgeRouter X (ER-X) 设备的 OEM（原始设备制造商）分区布局包含两个内核槽位，每个槽位大小为 3MB。  

从 OpenWrt 24.10 版本开始，由于其中包含的 Linux 6.6 内核映像尺寸增加，已不再适配原有的分区布局。因此，当用户升级到 OpenWrt 24.10 时，需要迁移到新的分区布局。  

PR [#15194](https://github.com/openwrt/openwrt/pull/15194) 引入了一种新的分区布局，允许内核映像最大可达 6MB：  

```sh
root@OpenWrt:/# cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00080000 00020000 "u-boot"
mtd1: 00060000 00020000 "u-boot-env"
mtd2: 00060000 00020000 "factory"
mtd3: 00600000 00020000 "kernel"
mtd4: 0f7c0000 00020000 "ubi"
```

本仓库中的脚本可用于将现有 21.02、22.03 或 23.05 版本的安装迁移到新的布局。  

它将为您的设备安装最新的 24.10 发布版本。我们希望这些脚本能被反向移植到 23.05 的下一个稳定版本中。不幸的是，由于内核映像过大，OpenWrt 24.10 无法再构建出厂映像（factory images）。因此，新的安装应该先刷入旧版本的出厂映像，然后按照以下说明直接迁移到 24.10 。  

## 迁移步骤 (Migration)

**警告：** 所有设置都将被擦除，请务必在操作前备份您的配置。  

1. 如果您仍在使用原厂操作系统 (stock OS)，请先安装 22.03[^1] 的出厂映像.  
2. （可选）如果您需要在没有互联网的情况下工作，请提前下载适用于您路由器型号的正确 24.10 sysupgrade.bin 文件并进行校验[^2]。  
3. 使用 SSH 登录到您的路由器。  
4. 将两个脚本都复制到路由器的 `/tmp/` 目录中，并确保它们分别命名为 `ubnt_erx_migrate.sh` 和 `ubnt_erx_stage2.sh` 。  
5. （可选）如果您预先下载了固件文件，请将其重命名并复制到路由器的 `/tmp/sysupgrade.img` 目录，步骤与复制脚本相同。  
6. 运行以下 shell 命令：  
	```sh
	cd /tmp
	chmod +x ubnt_erx_migrate.sh
	./ubnt_erx_migrate.sh
	```
7. 脚本将下载固件更新（如果需要）并进行校验，然后刷写新的内核和根文件系统（rootfs），最后重启设备。  
8. 设备将处于出厂默认的配置状态。  

**升级后恢复备份**  

1. 使用 SSH 登录路由器（用户名为 `root` ，无密码）。  
2. 检查兼容版本（compat version）是否已设置为 `2` ：`uci get system.@system[0].compat_version` 。  
3. 恢复备份文件（通过命令行或 Web 界面），如果未自动重启，请手动重启设备。  
4. 再次使用 SSH 登录路由器。  
5. 确保迁移成功完成，然后手动将兼容版本更新回 `2.0` ：  
	```sh
	uci set system.@system[0].compat_version=2.0
	uci commit
	```

## 快照版本（Snapshot）

您可以使用以下命令安装 24.10 的快照（snapshot）版本：  

```sh
SNAPSHOT=y ./ubnt_erx_migrate.sh
```

## 本地安装（Local Install）

您也可以自行编译 OpenWrt 快照版本并直接迁移到您自编译的版本：  

- 将您自编译的 `openwrt-ramips-mt7621-ubnt_edgerouter-x-squashfs-sysupgrade.bin` 和 `sha256sums` 文件托管在您的局域网 Web 服务器上，使其可以通过 `http://<host>/downloads-path/` 访问。  
- `http://<host>/downloads-path/` 必须支持自动索引，或至少返回一个包含类似 `href=http://<host>/downloads-path/<your-image-file-name>-squashfs-sysupgrade.bin` 链接的页面。  
- 在路由器上设置环境变量：`export TESTSITE="http://<host>/downloads-path/"`  
- 然后按照上述步骤继续操作。  

## TFTP 串口安装（TFTP Serial Installation）

另外，您也可以通过串口控制台直接安装新的固件：  

1. 按下 `<1>` 进入 U-Boot 菜单，选择 TFTP 安装。  
2. 启动 `openwrt-ramips-mt7621-ubnt_edgerouter-x-initramfs-kernel.bin`  
3. 然后运行 `sysupgrade -n -F openwrt-ramips-mt7621-ubnt_edgerouter-x-squashfs-sysupgrade.bin`  

[^1]: https://github.com/stman/OpenWRT-19.07.2-factory-tar-file-for-Ubiquiti-EdgeRouter-x
[^2]: https://openwrt.org/docs/guide-quick-start/verify_firmware_checksum
