# FortiGate gadgets

**本仓库中的工具仅用于安全研究目的，不应在生产环境中使用。**

## License

### FDS server

`fds_server.py` 是一个用于 FortiGate 的自定义授权服务器，可用于较旧版本，例如 FortiGate VM64 v7.4.1。与之前的方法相比，使用 FDS server 可以修改序列号（包括前缀）、增加 CPU 和内存数量，并且可以激活更多的 VDOM。目前，该方法仅适用于旧版本。

**使用方法**

以 FortiGate VM64 7.4.1（VMWARE）为例。

首先需要部署虚拟机并在 CLI 中完成网络接口的配置。然后需要在与 FortiGate 在同一网络的系统上启动 FDS server（确保 FortiGate 可以访问 FDS server 的 8890 端口）。

在 FortiGate 上执行以下命令：

config system central-management
set mode normal
set type fortimanager
set fmg <FDS 服务器的 IP 地址>
config server-list
edit 1
set server-type update rating
set server-address <FDS 服务器的 IP 地址>
end
set fmg-source-ip <FortiGate 的 IP 地址>
set include-default-servers disable
set vdom root
end

go
复制代码

运行 `license_old.py` 脚本以生成 License 文件，然后登录 FortiGate 的 Web 界面导入该 License 文件。

系统将自动重启。系统启动后，你应该能够在 FDS server 端看到类似输出，例如：

========================
[] Parsing data
[+] Magic: PUTF
[+] System version: 07004000
[+] Payload length: 363
[+] Header length: 64
[+] Time: 202505301704
[+] Header crc32: 0x946cdd64
[] Parsing obj
[+] Magic: FCPC
[+] Name: Command Object
[+] Payload length: 235
[+] System version: 07004000
[+] Payload crc32: 0x209e4867
[+] Header crc32: 0xaa2ece4
[+] Payload: b'Protocol=3.0|Command=VMSetup|Firmware=FGVM64-FW-7.04-2463|SerialNumber=FGVM32GVOVCLUK2G|Connection=Internet|Address=192.168.66.150:0|Language=en-US|TimeZone=-7|UpdateMethod=1|Uid=564d678fc9f2506bb8aebfde4052bbbd|VMPlatform=VMWARE\r\n\r\n\r\n'
[] Packing obj
[] Packing req
[*] Sending response

yaml
复制代码

登录 Web 界面。如果一切正常，将会进入配置向导。  
**不要注册 FortiCare 并禁用自动补丁升级。**

![](img/reg_wizard1.png)

![](img/reg_wizard2.png)

**缺点**

- FortiCare 支持缺失，因此系统无法接收 AntiVirus/IPS/Firmware 等更新。
- 部分功能可能无法正常工作（未测试）。

如果你遇到任何问题，请提交 Issue。

---

### 版本 > 7.4.1

**该部分暂未更新。**

对于较新版本，你需要先对 `flatkc` 和 `init` 进行补丁处理。请依照以下步骤：

导入 OVF 模板并启动系统，等待系统完成初始化

关闭虚拟机并移除第一块虚拟磁盘（2GB）

将该磁盘挂载到另一台 Linux 系统

挂载根分区（FORTIOS），提取 "flatkc" 和 "rootfs.gz" 文件并做好备份

运行：python3 decrypt.py -f rootfs.gz -k flatkc 以解密 rootfs.gz

解压 rootfs.gz 和 bin.tar.xz（需 root 权限）
gzip -d ./dec.gz
mkdir rootfs
cd rootfs && mv ../dec ./
sudo su
cpio -idmv < ./dec
rm -rf ./dec
xz -d ./bin.tar.xz && tar -xvf ./bin.tar && rm -rf ./bin.tar
cd .. && mv ./rootfs/bin/init ./

运行：python3 patch.py init 生成补丁 init

如果需要，可以添加其他文件（如 busybox）。然后重新压缩 rootfs：
chmod 755 ./init.patched && mv ./init.patched ./rootfs/bin/init
cd rootfs
tar -cvf bin.tar bin && xz bin.tar && rm -rf bin
find . | cpio -H newc -o > ../rootfs.raw && cd ..
cat ./rootfs.raw | gzip > rootfs.gz

运行：python3 patch.py flatkc 以补丁 flatkc

将 rootfs.gz 与 flatkc.patched 写回虚拟磁盘

将虚拟磁盘从 Linux 卸载并重新挂回原虚拟机

启动系统

yaml
复制代码

启动后，运行 `python3 license_new.py` 并导入生成的 `License.lic`。

> 注意：系统重启后可能需要重新配置网络接口 IP。

详见：https://wzt.ac.cn/2024/04/02/fortigate_debug_env2/

---

### VDOM license

需要先安装 libssl-dev。

编译：

gcc vdom.c -o vdom -lssl -lcrypto -lz

复制代码

运行：

./vdom FGVMPG0000000000 15

复制代码

导入 License：

execute upd-vd-license xxx

复制代码

如果看到：

Error: VDOM number (xxx) exceeds limit for this model

yaml
复制代码

说明你的基础 License 不支持更多 VDOM 数量。

---
