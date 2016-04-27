

# ADB 学习总结

ADB 全称是Android Debug Bridge，android设备和模拟器是通过ADB来和PC机进行通信的，可以认为ADB是2者之间通信的桥梁。ADB采用的是cs模式来进行数据通信。

## ADB 有三个部分组成
  
 1. ADB客户端（DDMS）
 2. ADB服务（运行在PC主机上 端口号5037）
 3. adbd 守护服务（运行在模拟器或者android设备上)



![http://77fzym.com1.z0.glb.clouddn.com/%E4%B8%8B%E8%BD%BD.png](http://77fzym.com1.z0.glb.clouddn.com/%E4%B8%8B%E8%BD%BD.png)

当用户开始一个ADB客户端程序时，客户端首先会检查是否已经有一个 ADB 服务进程在运行，如果没有，它会启动一个服务进程；服务进程启动后，会与一个本地TCP端口5037进行绑定，并监听从ADB客户端发送过来的命令（所有的客户端都使用端口5037与服务进程进行通信）。

接着，服务端进程会配置所有正在运行的模拟器或者设备的链接：通过扫描5555 - 5585（模拟器/设备使用的端口范围）之间的奇数端口，服务端进程可以找到相应的模拟器/设备，接着找到相应的ADB守护进程，并对该端口进行配置。注意，每个模拟器/设备都需要一对连续的端口 - 奇数端口号用于ADB连接，偶数端口号用于控制台连接。


## Android设备ADB授权的原理

用ADB调试Android设备时，首次连接时，会出现一个授权提示：

> error: device unauthorized. Please check the confirmation dialog on your device.

这时候，正常情况下，在手机上会出现一个提示框，让用户确认是否授权这个PC设备允许调试，你只需要点击确认就可以了！

**工作原理是什么？**

在PC机上首次启动 ADB.exe进程时，ADB会在C盘的当前用户的目录下生成一个".android"目录，生成一对密钥ADBkey(私钥)与ADBkey.pub(公钥)就存放在这个目录下。在电脑初次通过ADB和android设备通信的时候，PC端会把公钥(或者公钥的hash值)(fingerprint)发送给android设备，该公钥存放在android设备的"/data/misc/ADB/"目录下。当android设备的"/data/misc/ADB/"目录下已经存在该公钥时，就不发送这个文件，如果android设备没有这个文件的时候，就会发送PC上的公钥到android设备上的"/data/misc/ADB/"目录下，则会弹出提示框，让你确认是否允许这台机器进行ADB连接，当你点击了允许授权之后，android就会保存了这台PC的ADBkey.pub(公钥)；

参考 [http://blog.csdn.net/sowhat_ah/article/details/43307907](http://blog.csdn.net/sowhat_ah/article/details/43307907)

## ADB常用命令

> 查看已连接的设备
  adb devices

---

> 安装软件
adb install <path_to_apk>

---

> 拷贝文档到手机或者模拟器
adb push foo.txt /sdcard/foo.txt

---

> 拷贝手机文件至电脑上
adb pull <remote> <local>

---












 


 
    

