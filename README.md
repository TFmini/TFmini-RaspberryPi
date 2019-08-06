# TFmini-RaspberryPi  
**Note: also suitable for TFmini Plus**  

The example of using TFmini on Raspberry Pi 3. 中文版参考 [README_CN.md](/README_CN.md).  
   
Raspberry Pi has serial port, IO port and USB, so there are a lot of way to connect it with TFmini:

- connection via serial port (RXD0 and TXD0)
- connection via USB-serial adaptor (CP2102, CH341, etc.)  
- connection via simulative serial port from IO port (via pigpio, etc.)

Run console command `ls /dev` , Raspberry Pi 3 has the following identified serial ports (may be different on different devices):  


Peripherals | Raspberry Pi(/dev/) 
---------|----------
 Hardware serial port | ttyAMA0 
 Software serial port | ttyS0
 Arduino | ttyACM0, ttyACM1... 
 USB-serial adaptor(CP2102, CH341...) | ttyUSB0, ttyUSB1...

 Raspberry Pi 3 has own hardware serial port (PL011) and software serial port (mini UART). The hardware serial port is connected to bluetooth BT by default, and pins (RXD0 and TXD0) are also closed by default. The configurable hardware serial port is not connected to bluetooth and the pins can be configured to connect to hardware or software serial port.

 **Hardware serial port** has high precision as well as completed configuration, and even could read data from TFmini without verification. **Software serial port** should not skip verification while reading data from TFmini. **Simulative serial port** could have lower precision, so, except verification, it is better to add threshold detection to ensure data's correctness.   

 When choosing the programming language, according to [RPi GPIO Code Samples](https://elinux.org/RPi_GPIO_Code_Samples#pigpio_2) , C, C#, Ruby, Python, Java ... all kinds of languages can be used. Here we use Python as example, and any other requirements can be submitted in issues.  

## Install and Configure Raspbian
This section can be skipped if the RaspberrtPi system has been installed and configured.  

Download operaing system image: [Raspbian](https://www.raspberrypi.org/downloads/raspbian/). We choose RASPBIAN STRETCH WITH DESKTOP and, normally, the download should be completed quickly. What we download is a .zip file, thus decompress it to get the .img file. 

Download [Etcher](https://etcher.io/). It is used to burn the operating system image to SD/TF card and is the offical document [INSTALLING OPERATING SYSTEM IMAGES](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)'s recommanded burning method. We use a Class 10, 32GB TF card, connect the card to PC via card reader, open Etcher, choose the .img file decompressed at the step above and then choose the TF card, finally start burning. The burning should take several minutes and we connect the TF card to Raspberry Pi after it is done.   

Connect the Raspberry Pi to monitor, mouse and keyboard. Power up and then enter operating system.   

It is recommanded to close bluetooth by click the bluetooth icon in taskbar.    

Connect Raspberry Pi to a WiFi network, open the console and enter:

```
sudo raspi-config
```  

Choose 5. Interfacing-Options -> P2 SSH -> Yes, to enable SSH.  

Choose 5. Interfacing-Options -> P3 VNC -> Yes, to enable VNC, so that we can use remote graphic interface.  

Choose 5. Interfacing-Options -> P6 Serial -> Yes, to enable serial port.  

Then we can connect our PC to Raspberry Pi 3 via SSH, VNC or serial port. We choose SSH as example, so we should keep the Raspberry Pi within the WiFi range and get Raspberry Pi's IP address.

Log in to router(it should have a default IP address), or use [Angry IP Scanner](http://angryip.org/download/), to search Raspberry Pi's IP address. It should be at the range of 192.168.1.1~192.168.1.255 by default. Mine is 192.168.1.231.   

Open [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/), use SSH connection, enter IP address and port 22, press open. It will prompt that PuTTY Security Alert at the first time, choose Yes. Username is pi, and password is raspberry by default:  

![login](/Assets/login.png)  

Update first:

```
sudo apt-get update && sudo apt-get upgrade -y
sudo rpi-update
sudo reboot
```  

Reconnect to the Raspberry Pi via SSH after reboot (IP address may change).

If connect PC to Raspberry Pi via serial port, following information may be printed on the console:  

```
Raspbian GNU/Linux 9 raspberrypi ttyS0
raspberrypi login: 
``` 

It indicate that the serial port is used to output console information. It is not what we intend to (console is linked to serial0 by default, baud rate 115200), we want to use it connect to peripheral device, so it is need to disable console:

```
sudo nano /boot/cmdline.txt

dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait

sudo reboot
```

For Raspberry Pi 3, although the serial port is freed, but it is an unreliable mini UART. PL011 hardware serial port is a better choose, so let us check the mapping:

```
ls -l /dev
#serial0 -> ttyS0
#serial1 -> ttyAMA0
```

Change the mapping:  

```
sudo nano /boot/config.txt
```  

Add a line:  

```
dtoverlay=pi3-miniuart-bt
```  

Reboot: 

```
sudo reboot
```

Check the mapping again:  

```
ls -l /dev
#serial0 -> ttyAMA0
#serial1 -> ttyS0
```

In brief, we close the console linked to serial port, and then map the hardware serial port to TXD0, RXD0. Now we can use serial port via /dev/ttyAMA0.

Raspberry Pi is connected to PC via WiFi in the same LAN. We use SSH command for connection, VNC for graphic interface, Filezilla for files transmission (sftp://[Raspberry Pi's IP address], username is pi, password is raspberry). If we configure Samba well, we can edit the code on Raspberry Pi as editing local files on PC, and VS Code is a very convenient editor. You can search for more information if you are intereted in, we won't introduce them here due to limited space (and time).


## Connect TFmini to Raspberry Pi's Serial Port  

Let's check Raspberry Pi's GPIO first:

![Rpi3_GPIO](/Assets/RPi3_GPIO.png)  

Connection is as follow:  

Raspberry Pi 3 | TFmini
---------|----------
 +5V | 5V(RED)
 GND | GND(BLACK) 
 TXD0 | RX(WHITE) 
 RXD0 | TX(GREEN)   

In fact, we don't need to send command to TFmini, we only need to receive data from TFmini, so TXD0 can be vacant or connected to another device.

tfmini.py code: 

```Python
# -*- coding: utf-8 -*
import serial

ser = serial.Serial("/dev/ttyAMA0", 115200)

def getTFminiData():
    while True:
        count = ser.in_waiting
        if count > 8:
            recv = ser.read(9)
            ser.reset_input_buffer()
            if recv[0] == 'Y' and recv[1] == 'Y': # 0x59 is 'Y'
                low = int(recv[2].encode('hex'), 16)
                high = int(recv[3].encode('hex'), 16)
                distance = low + high * 256
                print(distance)
                


if __name__ == '__main__':
    try:
        if ser.is_open == False:
            ser.open()
        getTFminiData()
    except KeyboardInterrupt:   # Ctrl+C
        if ser != None:
            ser.close()

```

Enter `python tfmini.py` to run and we can see the results. Note that it won't work with python3.  

tfmini_23.py code:  

```Python
# -*- coding: utf-8 -*
import serial
import time

ser = serial.Serial("COM12", 115200)

def getTFminiData():
    while True:
        #time.sleep(0.1)
        count = ser.in_waiting
        if count > 8:
            recv = ser.read(9)   
            ser.reset_input_buffer() 
            # type(recv), 'str' in python2(recv[0] = 'Y'), 'bytes' in python3(recv[0] = 89)
            # type(recv[0]), 'str' in python2, 'int' in python3 
            
            if recv[0] == 0x59 and recv[1] == 0x59:     #python3
                distance = recv[2] + recv[3] * 256
                strength = recv[4] + recv[5] * 256
                print('(', distance, ',', strength, ')')
                ser.reset_input_buffer()
                
            if recv[0] == 'Y' and recv[1] == 'Y':     #python2
                lowD = int(recv[2].encode('hex'), 16)      
                highD = int(recv[3].encode('hex'), 16)
                lowS = int(recv[4].encode('hex'), 16)      
                highS = int(recv[5].encode('hex'), 16)
                distance = lowD + highD * 256
                strength = lowS + highS * 256
                print(distance, strength)
            
            # you can also distinguish python2 and python3: 
            #import sys
            #sys.version[0] == '2'    #True, python2
            #sys.version[0] == '3'    #True, python3


if __name__ == '__main__':
    try:
        if ser.is_open == False:
            ser.open()
        getTFminiData()
    except KeyboardInterrupt:   # Ctrl+C
        if ser != None:
            ser.close()
```   
 
This works with python2 and python3. 

## Connect TFmini to Raspberry Pi's Other IO Ports  

According to the GPIO image of the section above, we connect TX(GREEN) to GPIO23, while RX(WHITE) is not necessary and can be vacant. Using simulative serial port to receive data is not reliable, so we add verification of checksum and threshold filter in the code to avoid data error.

tfmini_ss.py code:  

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

Run:  

```Shell
# pigpio scripts require that the pigpio daemon be running.
sudo pigpiod
python tfmini_ss.py
```  

Although simulative serial port is not reliable, but we have to use this method in some cases. For example, the only serial port of Raspberry Pi could be used to connect to another device except TFmini, so we can use any other IO to connect to TFmini. Moreover, in theory, every IO can be used to connect to TFmini, thus we can connect multiple TFmini to one Raspberry Pi to satisfy some applications' needs.
  
Using Raspberry Pi's USB is another way. We can connect multiple TFmini to Raspberry Pi via USB-serial adaptor and USB Hub.  
