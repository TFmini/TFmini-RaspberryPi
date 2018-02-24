# TFmini-RaspberryPi
TFmini在树莓派3上的例子.  
树莓派既有引出串口, IO口, 又有USB, 所以可以有很多种方法连接TFmini:  

- 树莓派引出串口(RXD0和TXD0)
- USB转串口(CP2102, CH341...)  
- 其他引出IO口模拟串口(通过pigpio等)

终端输入 `ls /dev` ,树莓派3串口的识别关系如下(可能有出入?):  


外设 | 树莓派(/dev/) 
---------|----------
 硬件串口 | ttyAMA0 
 软件串口 | ttyS0
 Arduino | ttyACM0, ttyACM1... 
 USB转串口(CP2102, CH341...) | ttyUSB0, ttyUSB1...

 树莓派3自带硬件串口(PL011)和软件串口(mini UART), 硬件串口默认连接蓝牙BT, 引出串口(RXD0和TXD0)默认也是关闭的, 可软件配置硬件串口不和蓝牙相连, 引出串口可配置连接硬件串口还是软件串口.  

 **硬件串口**精度高, 配置全, 连接TFmini甚至可能不需要校验. **软件串口**连接TFmini校验不能少. **模拟串口**精度可能更差, 除了校验, 最好加上阈值判定来保证数值正确性.   

 编程语言上, 参考 [RPi GPIO Code Samples](https://elinux.org/RPi_GPIO_Code_Samples#pigpio_2)来看, C, C#, Ruby, Python, Java...各种语言应该都是可以的. 这里我们选择Python作为例子. 有其他需求可以在issues里面提.  

## 安装和配置Raspbian
已经安装和配置好树莓派系统的略过本节.  

下载树莓派的系统: [Raspbian](https://www.raspberrypi.org/downloads/raspbian/), 这里选的是 RASPBIAN STRETCH WITH DESKTOP, 下载Torrent, 然后用百度云离线下载, 一般会瞬间完成, 然后在开始下载, 非会员的可以用每天的免费加速, 一般很快就可以下完.  下完后是.zip, 解压出里面的.img文件. 

下载 [Etcher](https://etcher.io/), 用于把上面下载的系统烧录到SD/TF卡, 这是官方文档 [INSTALLING OPERATING SYSTEM IMAGES](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)推荐的烧录方式. 这里选的是Class 10, 32GB的TF卡, 用读卡器连到电脑上, 打开Etcher, 选择上面的 .img文件, 然后选择TF卡, 最后烧录, 几分钟后就完成了, 完成后插到树莓派上即可.   

新系统坑的一匹, SSH默认关闭, 硬件串口默认连接BT, 调试串口默认关闭?(默认不会通过引出串口输出Linux调试信息?), 不连显示器还真不知道怎么搞了...  

接上显示器, 鼠标, 键盘. 上电, 进入系统.   

~~点击上面任务栏的蓝牙图标, 关闭蓝牙.~~    

连上WiFi, 打开终端, 输入:  

```
sudo raspi-config
```  

选择 5. Interfacing-Options -> P2 SSH -> Yes, 使能SSH.  

选择 5. Interfacing-Options -> P3 VNC -> Yes, 使能VNC, 这个可以远程图形界面.  

选择 5. Interfacing-Options -> P6 Serial -> Yes, 使能串口.  

这样就可以把 借来的读卡器/网线/鼠标/键盘/显示器/HDMI线一大堆东西还回去了.  至于树莓派, 放到小车上, 飞行器上, 办公室/实验室其他地方都是可以的, 只要同一个局域网的WiFi覆盖到就可以.  

然后我们就可以在电脑上通过 SSH/VNC/串口 的任意一种连接树莓派3了, 这里我们选择SSH, 所以首先要获取树莓派的IP地址.  

登录路由器(默认192.168.1.1), 或者使用 [Angry IP Scanner](http://angryip.org/download/), 查找 raspberrypi 的IP, 默认应该在 192.168.1.1~192.168.1.255 之间. 我这里是192.168.1.231.   

打开 [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/), 使用SSH连接, 输入IP, 端口22, open,  第一次会弹出PuTTY Security Alert, 根据提示选择Yes. 用户名 pi, 密码 raspberry:  

![login](/Assets/login.png)  

先更新/升级点东西:  

```
sudo apt-get update && sudo apt-get upgrade -y
sudo rpi-update
sudo reboot
```  

重启后重新连接SSH(IP地址可能会变).   

如果连接串口的话, 可能就会看到引出串口打印一些类似下面的信息:  

```
Raspbian GNU/Linux 9 raspberrypi ttyS0
raspberrypi login: 
``` 

表明这个时候的串口用作 console(控制台) 了, 这不是我们想要的(console默认连接的是serial0, 115200), 我们只是想连接外设用, 所以需要disable console:  

```
sudo nano /boot/cmdline.txt

dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait

sudo reboot
```

对于树莓派3而言, 虽然串口已经解放出来了, 但使用的是不稳定的mini UART, 我们用 PL011 硬件串口更好, 先看一下映射关系:  

```
ls -l /dev
#serial0 -> ttyS0
#serial1 -> ttyAMA0
```

改变映射关系:  

```
sudo nano /boot/config.txt
```  

增加一行:  

```
dtoverlay=pi3-miniuart-bt
```  

重启: 

```
sudo reboot
```

看一下映射关系:  

```
ls -l /dev
#serial0 -> ttyAMA0
#serial1 -> ttyS0
```

简言之, 我们关闭了串口的console, 然后把硬件串口映射到TXD0, RXD0. 使用的时候引出串口是 /dev/ttyAMA0.  

树莓派通过WiFi和PC在同一个局域网中, 连接方式上, 命令行用SSH, 图形界面用VNC, 文件传输可以使用Filezilla(sftp://树莓派IP, 用户名pi, 密码raspberry). 如果我们配置Samba, 在电脑上就可以像编辑本地文件一样编辑树莓派的代码, 用VS Code之类的编辑器就非常方便了, 感兴趣的可以自己搜索, 这里限于篇幅就略了.   


## 连接TFmini 到树莓派引出串口  

先看下树莓派的IO:  

![Rpi3_GPIO](/Assets/RPi3_GPIO.png)  

连接关系如下:  

Raspberry Pi 3 | TFmini
---------|----------
 +5V | 5V(红)
 GND | GND(黑) 
 TXD0 | RX(白) 
 RXD0 | TX(绿)   

 实际上, 我们这里没有给TFmini发命令, 只用接收TFmini发来的数据即可, 所以, TXD0可以不接或者连接其他外设.  

tfmini.py 文件如下: 

```Python
# -*- coding: utf-8 -*
import serial

ser = serial.Serial("/dev/ttyAMA0", 115200)

def getTFminiData():
    while True:
        count = ser.in_waiting
        if count > 8:
            recv = ser.read(9)
            if recv[0] == 'Y' and recv[1] == 'Y': # 0x59 is 'Y'
                low = int(recv[2].encode('hex'), 16)
                high = int(recv[3].encode('hex'), 16)
                distance = low + high * 256
                print(distance)
                ser.reset_input_buffer()


if __name__ == '__main__':
    try:
        if ser.is_open == False:
            ser.open()
        getTFminiData()
    except KeyboardInterrupt:   # Ctrl+C
        if ser != None:
            ser.close()

```

`python tfmini.py` 运行, 即可看到结果, 注意这里用python3不会工作.   

## 连接TFmini 到树莓派其他IO口  

参考上节中树莓派的IO口图, 我们把 TX(绿) 连到GPIO23上, RX(白)这里没有用上, 可以悬空. 模拟串口数据接收不太稳定, 所以我们代码中加上了校验checksum和阈值筛选, 尽量避免数据错误.  

tfmini_ss.py 代码如下:  

```Python
# -*- coding: utf-8 -*
import pigpio
import time

RX = 23

pi = pigpio.pi()
pi.set_mode(RX, pigpio.INPUT)
pi.bb_serial_read_open(RX, 115200) 

def getTFminiData():
	while True:
		#print("#############")
		time.sleep(0.05)	#change the value if needed
		(count, recv) = pi.bb_serial_read(RX)
		if count > 8:
			for i in range(0, count-9):
				if recv[i] == 89 and recv[i+1] == 89: # 0x59 is 89
					checksum = 0
					for j in range(0, 8):
						checksum = checksum + recv[i+j]
					checksum = checksum % 256
					if checksum == recv[i+8]:
						distance = recv[i+2] + recv[i+3] * 256
						strength = recv[i+4] + recv[i+5] * 256
						if distance <= 1200 and strength < 2000:
							print(distance, strength) 
						#else:
							# raise ValueError('distance error: %d' % distance)	
						#i = i + 9

if __name__ == '__main__':
	try:
		getTFminiData()
	except:  
		pi.bb_serial_read_close(RX)
		pi.stop()
 
```

运行:  

```Shell
# pigpio scripts require that the pigpio daemon be running.
sudo pigpiod
python tfmini_ss.py
```  

模拟串口尽管进度较差, 但好处太多, 比如树莓派就那一个宝贝的引出串口, 可能轮不到连接TFmini, 用模拟串口就可以随意找一个IO连接TFmini的TX, 这样就留出了宝贵的IO口, 而且, 有了模拟串口, 每个IO口都可以连接一个TFmini, 同时连接多个TFmini, 再也不用担心串口不够用的问题.   

当然树莓派的USB毕竟也不是摆设, 连接多个TFmini也可以用通过USB转串口(CP2102, CH341等)插到USB上去, 再多还有USB Hub可以用. 