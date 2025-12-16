**参考文献** 
1. [在WSL2中的Vivado如何连接到FPGA开发板 - fmq03 - 博客园](https://www.cnblogs.com/fmq03/articles/18806644))
2. [连接 USB 设备 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb#attach-a-usb-device.)
3. [WSL下的FPGA指南 - Yiipu/HUST-HardwareDesign-RISCv GitHub Wiki](https://github-wiki-see.page/m/Yiipu/HUST-HardwareDesign-RISCv/wiki/WSL%E4%B8%8B%E7%9A%84FPGA%E6%8C%87%E5%8D%97)

**运行环境**
`windows11 | wsl2 | ubuntu24.04 | vivado 2022.1 | z730v1.3`

# 1. 安装 usbipd (windows中)
管理员权限下在`powershell`中运行
`winget install --interactive --exact dorssel.usbipd-win` 
从而安装`usbipd`包
# 2. 安装 cable 驱动 (wsl中)
前置：你需要先把vivado装在wsl里，然后找到vivado根目录
本文档使用的vivado版本是2022.1

hints: `sudo find / -name vivado` 找到根目录 `${vivado_root}`

然后找到`install_drivers`可执行文件
```bash
TypeC-Fuxuan@localhost:~/projects/cpu/vivado$ find . -name install_drivers
./Vitis/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers
./Vitis/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers
./Vivado/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers
./Vivado/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers
```
第4个路径`./Vivado/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers`是正确的

然后
```bash
cd ./Vivado/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers/
sudo ./install_drivers
```
如果出现
```bash
INFO: Digilent Return code = 0
INFO: Xilinx Return code = 0
INFO: Xilinx FTDI Return code = 0
INFO: Return code = 0
INFO: Driver installation successful.
```
那就是安装好了，请转到Section 3

**[可能的bug]**但是也有可能出现这种情况
```bash
./Vivado/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers: 43: ./install_digilent.sh: not found
./Vivado/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers: 45: ./setup_xilinx_ftdi: not found
./Vivado/2022.1/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers: 47: ./setup_pcusb: not found
--real rc=127

ERROR: Driver installation failed.

INFO: Digilent Return code = 127
INFO: Xilinx Return code = 1
INFO: Xilinx FTDI Return code = 127
INFO: Return code = 1
```
那就是你没`cd`进去对应的文件夹 (踩坑x1，还是要说明白一点)

# 3. 映射usb端口 
## 3.1 准备环境 (wsl中)
先装个`usbutils`
`sudo apt install usbutils -y`

## 3.2 确认BUSID (windows中)
先把USB-UART的type-c供电插上(USB口插到电脑上别插插座 (踩坑x2))，还有JTAG也接上.
然后用`powershell` 
```powershell
> usbipd list
Connected:
BUSID  VID:PID    DEVICE                       STATE
1-3    046d:c52f  USB 输入设备                  Not shared
1-4    1a86:7523  USB-SERIAL CH340 (COM6)      Not shared
1-6    04f2:b7b8  Integrated Camera            Not shared
1-9    0403:6014  Digilent USB Device          Not shared
1-12   048d:c992  USB 输入设备                  Not shared
1-14   0bda:5852  Realtek Bluetooth Adapter    Not shared
```
然后你会看到 
`1-4    1a86:7523  USB-SERIAL CH340 (COM6) Not shared`是USB-UART
`1-9    0403:6014  Digilent USB Device     Not shared`是JTAG
如果不确定 先拔了JTAG线，看看哪个没了
然后再拔USB-UART供电，再看看哪个没了

总之能够记录到对应的`BUSID`

## 3.3 绑定USB
首先，需要保证开**起码一个wsl的bash**，不然没办法attach到wsl, 如果绑定USB失败, 修复方法见 [3.6](#helper)
请务必按照顺序，先绑定USB-UART，再绑定JTAG
若单绑定JTAG，这个玩意会因为在其之后绑定USB-UART而直接断联，亦或是短暂的一两秒内被wsl杀了
(具体可以参考Section4)

```powershell
# powershell
> usbipd bind -b 1-4  # 先配置USB-UART
> usbipd bind -b 1-9  # 再配置JTAG

> usbipd list         # 看看结果
1-4    1a86:7523  USB-SERIAL CH340 (COM6)      Shared
1-9    0403:6014  Digilent USB Device          Shared


> usbipd attach --wsl -b 1-4 # 先配置USB-UART
usbipd: info: Using WSL distribution 'ubuntu-24.04' to attach; the device will be available in all WSL 2 distributions.
usbipd: info: Detected networking mode 'mirrored'.
usbipd: info: Using IP address 127.0.0.1 to reach the host

> usbipd attach --wsl -b 1-9 # 再配置JTAG
usbipd: info: Using WSL distribution 'ubuntu-24.04' to attach; the device will be available in all WSL 2 distributions.
usbipd: info: Detected networking mode 'mirrored'.
usbipd: info: Using IP address 127.0.0.1 to reach the host

> usbipd list         # 看看结果
1-4    1a86:7523  USB-SERIAL CH340 (COM6)      Attached
1-9    0403:6014  Digilent USB Device          Attached
```
然后就可以回去wsl那边了
检查一下`/dev` 里应该会有两个文件 `/dev/ttyUSB0` 和 `/dev/ttyUSB1`

然后可以在wsl中，检测一下绑定是否成功
```bash
# bash
$ dmesg | grep -i 'ftdi\|usbserial\|tty'
[ 4559.103339] usb 1-1: ch341-uart converter now attached to ttyUSB0
[ 4562.897296] ftdi_sio 1-2:1.0: FTDI USB Serial Device converter detected
[ 4562.898624] usb 1-2: FTDI USB Serial Device converter now attached to ttyUSB1
```
那么就能看到，`ttyUSB0`给到了USB-UART, `ttyUSB1`给到了FTDI(JTAG)
这个顺序貌似不是固定的？那不归我管

## 3.4 烧板
那么经过优化，就不需要考验手速了(具体故事可见[4](#old))
我们将通过JTAG烧入板子，执行`vivado -mode batch -source program_device.tcl`有两种情况
### 3.4.1 成功(摘取`open_hw_target`之后的内容)
```bash
# bash
$ pwd
~/2025-fall-yatcpu-repo/lab/vivado/z710v1.3

$ vivado -mode batch -source program_device.tcl

# 以上省略一万字...
# open_hw_target
INFO: [Labtoolstcl 44-466] Opening hw_target localhost:3121/xilinx_tcf/Digilent/210251A08870
# current_hw_device [get_hw_devices xc7z010_1]
# refresh_hw_device -update_hw_probes false [lindex [get_hw_devices xc7z010_1] 0]
INFO: [Labtools 27-1435] Device xc7z010 (JTAG device index = 1) is not programmed (DONE status = 0).
# set_property PROGRAM.FILE {./rv-z710v1.3-20/rv-z710v1.3-20.runs/impl_1/design_1_wrapper.bit} [get_hw_devices xc7z010_1]
# program_hw_devices [get_hw_devices xc7z010_1]
INFO: [Labtools 27-3164] End of startup status: HIGH
# refresh_hw_device [lindex [get_hw_devices xc7z010_1] 0]
INFO: [Labtools 27-1434] Device xc7z010 (JTAG device index = 1) is programmed with a design that has no supported debug core(s) in it.
# close_hw_target
INFO: [Labtoolstcl 44-464] Closing hw_target localhost:3121/xilinx_tcf/Digilent/210251A08870
INFO: [Common 17-206] Exiting Vivado at Wed Dec  3 00:00:39 2025...
```
### 3.4.2 失败
```bash
# bash
$ vivado -mode batch -source program_device.tcl

****** Vivado v2022.1 (64-bit)
  **** SW Build 3526262 on Mon Apr 18 15:47:01 MDT 2022
  **** IP Build 3524634 on Mon Apr 18 20:55:01 MDT 2022
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.

source program_device.tcl
# open_hw_manager
# connect_hw_server -allow_non_jtag
INFO: [Labtools 27-2285] Connecting to hw_server url TCP:localhost:3121
INFO: [Labtools 27-3415] Connecting to cs_server url TCP:localhost:3042
INFO: [Labtools 27-3414] Connected to existing cs_server.
ERROR: [Labtoolstcl 44-494] There is no active target available for server at localhost.
 Targets(s) ", jsn-JTAG-SMT2-210251A08870" may be locked by another hw_server.
INFO: [Common 17-206] Exiting Vivado at Wed Dec  3 00:42:30 2025...
```
为什么会失败，这是因为JTAG掉了，修复方法见 [3.6](#helper)

总之，烧完板子冒绿光就对了

## 3.5 读取串口数据
还记得之前我们提到`ttyUSB0`对应的是USB-UART吗，读结果从这个口读
先装个screen
`sudo apt install screen -y`
然后打开串口
`sudo screen /dev/ttyUSB0 115200`
会有两种结果
第一种进入了空白界面，没毛病
第二种显示`[screen is terminating]`或者是`Cannot exec '/dev/ttyUSB0': No such file or directory`, 修复方法见 [3.6](#helper)

这个时候你就可以打开PL_SW1(向上推)，可以看到红灯闪烁，CPU CLOCK在走
然后再将PL_SW2关闭再打开(向下推再向上)，就能看到screen里面有结果了
```bash
Never gonna give you up~ Never gonna let you down~
Never gonna run around and~ dessert you~
```
Congratulations! 这就搞好了
如果要退出，关掉bash就好，但是要注意，这个进程是没被杀掉的
意味着如果再次`sudo screen /dev/ttyUSB0 115200`打开，会报`[screen is terminating]`
同样见 [3.6](#helper)

但是如果你进到了空白界面，把两个bottom按照规则推了，又试了好几次，还是没有反应
见 [3.6](#helper)
## 3.6 一些问题的修复<a id="helper"></a>
### 3.6.1 JTAG或USB-UART掉了
首先先确认是掉了
```bash
# bash
$ dmesg | grep -i 'ftdi\|usbserial\|tty'
# 这个意味着JTAG掉了
[ 7875.708315] ftdi_sio ttyUSB1: FTDI USB Serial Device converter now disconnected from ttyUSB1
[ 7875.708329] ftdi_sio 1-2:1.0: device disconnected

# 这个意味着USB-UART掉了
[ 7912.438181] ch341-uart ttyUSB0: ch341-uart converter now disconnected from ttyUSB0
```
办法很简单，重新attach就可以
```powershell
# powershell
# 先detach (顺序不重要)
> usbipd detach -b 1-9
> usbipd detach -b 1-4

# 再attch (顺序重要,详见3.4)
> usbipd attach -b 1-4    # 先USB-UART
> usbipd attach -b 1-9    # 后JTAG
 
# 完事
```

### 3.6.2 `[screen is terminated]`及`screen`看不到输出等相关问题
首先确认占用情况，在bash中运行
`lsof /dev/ttyUSB0`
有三种情况
1. `lsof: status error on /dev/ttyUSB0: No such file or directory`那是**USB-UART掉了**，看上面那个修复

2. 有相关的进程在吃着
```bash
COMMAND   PID    USER            FD   TYPE DEVICE SIZE/OFF NODE NAME
screen    73520  TypeC-Fuxuan    5u   CHR  188,1  0t0      730  /dev/ttyUSB0
  ```
  那就是前面有个进程把设备占用了
  `kill <pid>`即可，此处是73520

3. 如果什么都没有，那请往后看

接下来，检查是不是选错了设备，如果接到了JTAG上，也不会有输出
试试另外一个device
`screen <other_device> 115200`
然后再把开关开开
那么你应该能看到输出，问题应该被解决了

### 3.6.3 `usb`端口映射失败
一看就是不细读文档，我猜你是这个问题
```powershell
usbipd attach -b 1-4
usbipd: error: There is no WSL 2 distribution running; keep a command prompt to a WSL 2 distribution open to leave it running.
```
不打开一个wsl的bash, 就没有正在运行的WSL 2 Distribution!!!
吃我一拳! ===========3

### 3.6.4 以上的问题都不是
(O_o)?? 你说还在往下看？那就要靠你自己了
加油！
``
## 4. 旧版烧板原文<a id="old"></a>
**[更新于 25.12.3]**
之前实验课上没有意识到JTAG和USB-UART是分离的，读了半天JTAG都读不出打印结果
而且单独绑定JTAG，WSL会在很短时间内把这个设备disconnect，因而还考验手速
总之我保留了下面的内容，好让你知道你吃的是大肠

**[以下是旧版原文]**
烧板要利用JTAG烧进去
这需要一定的手速
先在powershell中，将USB接口给到WSL
中间插入`usbipd list`明确正确状态
```powershell
# 注意是 JTAG 的 BUSID, 依据上文此处应为 1-9
> usbipd bind -b 1-9

> usbipd list
# 应是:
1-9    0403:6014  Digilent USB Device   Shared

> usbipd attach --wsl -b 1-9
usbipd: info: Using WSL distribution 'ubuntu-24.04' to attach; the device will be available in all WSL 2 distributions.
usbipd: info: Detected networking mode 'mirrored'.
usbipd: info: Using IP address 127.0.0.1 to reach the host.

> usbipd list
# 应是:
1-9    0403:6014  Digilent USB Device   Attached
```
别高兴的太早，此时要迅速输入烧板命令
`vivado -mode batch -source ./program_device.tcl`

然后会有两种结果
1. 成功(摘取`open_hw_target`之后的内容)
```bash
$ vivado -mode batch -source program_device.tcl

# 以上省略一万字...
# open_hw_target
INFO: [Labtoolstcl 44-466] Opening hw_target localhost:3121/xilinx_tcf/Digilent/210251A08870
# current_hw_device [get_hw_devices xc7z010_1]
# refresh_hw_device -update_hw_probes false [lindex [get_hw_devices xc7z010_1] 0]
INFO: [Labtools 27-1435] Device xc7z010 (JTAG device index = 1) is not programmed (DONE status = 0).
# set_property PROGRAM.FILE {./rv-z710v1.3-20/rv-z710v1.3-20.runs/impl_1/design_1_wrapper.bit} [get_hw_devices xc7z010_1]
# program_hw_devices [get_hw_devices xc7z010_1]
INFO: [Labtools 27-3164] End of startup status: HIGH
# refresh_hw_device [lindex [get_hw_devices xc7z010_1] 0]
INFO: [Labtools 27-1434] Device xc7z010 (JTAG device index = 1) is programmed with a design that has no supported debug core(s) in it.
# close_hw_target
INFO: [Labtoolstcl 44-464] Closing hw_target localhost:3121/xilinx_tcf/Digilent/210251A08870
INFO: [Common 17-206] Exiting Vivado at Wed Dec  3 00:00:39 2025...
```
2. 失败
```bash
$ vivado -mode batch -source program_device.tcl

****** Vivado v2022.1 (64-bit)
  **** SW Build 3526262 on Mon Apr 18 15:47:01 MDT 2022
  **** IP Build 3524634 on Mon Apr 18 20:55:01 MDT 2022
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.

source program_device.tcl
# open_hw_manager
# connect_hw_server -allow_non_jtag
INFO: [Labtools 27-2285] Connecting to hw_server url TCP:localhost:3121
INFO: [Labtools 27-3415] Connecting to cs_server url TCP:localhost:3042
INFO: [Labtools 27-3414] Connected to existing cs_server.
ERROR: [Labtoolstcl 44-494] There is no active target available for server at localhost.
 Targets(s) ", jsn-JTAG-SMT2-210251A08870" may be locked by another hw_server.
INFO: [Common 17-206] Exiting Vivado at Wed Dec  3 00:42:30 2025...
```
这个地方找不到 是因为你的手速太慢，一般来说这个USB attached后 过个几秒没有信息就被wsl杀掉了
