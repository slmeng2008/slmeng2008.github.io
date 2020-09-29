## usb gadget configfs 的使用

1. **挂载**

> mount -t configfs none /sys/kernel/config

挂载路径自定，上面命令挂载在  /sys/kernel/config 目录下

挂载后的目录结构如下，configfs自动生成usb_gadget目录
>config
>
>| ---- usb_gadget

2. 创建 gadget目录

>mkdir -p /sys/kernel/config/usb_gadget/my_gadget

挂载后的目录结构如下，都是usb描述符所需的一些东西

>config
>
>|----usb_gaget
>
>​				|----UDC
>
>​				|----bMaxPacketSize0
>
>​				|----bcdDevice
>
>​				|----bcdUSB
>
>​				|----bDeviceClass
>
>​				|----bDeviceProtocol
>
>​				|----bDeviceSubClass
>
>​				|----idProduct
>
>​				|----idVendor
>
>​				|----*configs*
>
>​				|----*functions*
>
>​				|----*strings*
>
>​				.


3. 创建字符串描述符等，写入USB VID PID等等相关值,具体意义需要参考USB spec

>mkdir -p /sys/kernel/config/usb_gadget/my_gadget/strings/0x409
>mkdir -p /sys/kernel/config/usb_gadget/my_gadget/configs/c.1/strings/0x409
>echo 0x1234 > /sys/kernel/config/usb_gadget/my_gadget/idVendor
>echo 0x0310 > /sys/kernel/config/usb_gadget/my_gadget/bcdDevice
>echo 0x0200 > /sys/kernel/config/usb_gadget/my_gadget/bcdUSB
>echo "1111" > /sys/kernel/config/usb_gadget/my_gadget/strings/0x409/serialnumber
>echo "slmeng" > /sys/kernel/config/usb_gadget/my_gadget/strings/0x409/manufacturer
>echo "UVC" > /sys/kernel/config/usb_gadget/my_gadget/strings/0x409/product
>echo 500 > /sys/kernel/config/usb_gadget/my_gadget/configs/c.1/MaxPower
>echo 0x5678 > /sys/kernel/config/usb_gadget/my_gadget/idProduct

之后的目录结构

>config

>|----usb_gaget
>
>​				|----UDC
>
>​				|----bMaxPacketSize0
>
>​				|----bcdDevice
>
>​				|----bcdUSB
>
>​				|----bDeviceClass
>
>​				|----bDeviceProtocol
>
>​				|----bDeviceSubClass
>
>​				|----idProduct
>
>​				|----idVendor
>
>​				|----*configs*
>
>​						|----*c.1*
>
>​								|----*strings*
>
>​										|----*0x409*
>
>​												|----configuration
>
>​								|----MaxPower
>
>​								|----bmAttributes
>
>​				|----*functions*
>
>​				|----*strings*
>
>​						|----*0x409*
>
>​									|----manufacturer
>
>​									|----product
>
>​									|----serialnumber


4. 以上是设备（gadget）和配置（config）相关的设置，具体到要实现某些功能需要配置接口（function）。比如mass storage、hid、uvc等或者是复合设备。

   例如hid uvc复合设备就要添加两个function。 下面已hid设备为例
> mkdir  /sys/kernel/config/usb_gadget/my_gadget/functions /hid.test0

ls查看目录下面生成的项目，这里是每种设备生成项目是不同的，上述命令有一定的命名规则例如：hid.xxx  rndis.xxx 等。具体需要参考Documentation/ABI/testting/configfs-usb-gadget\* 
>ls /sys/kernel/config/usb_gadget/my_gadget/functions/hid.test0/
>dev  protocol  report_desc  report_length  subclass

 hid相关文档为Documentation/ABI/testting/configfs-usb-gadget-hid
>
>```
>	What:		/config/usb-gadget/gadget/functions/hid.name
>	Date:		Nov 2014
>	KernelVersion:	3.19
>	Description:
>		The attributes:
>		protocol	- HID protocol to use
>		report_desc	- blob corresponding to HID report descriptors
>				except the data passed through /dev/hidg<N>
>		report_length	- HID report length
>		subclass	- HID device subclass to use
>					
>```

从可以看到命名规则和该function目录下生成的项目的意义。


5. 具体实现一个鼠标功能
> echo 2 > /sys/kernel/config/usb_gadget/my_gadget/functions/hid.test0/protocol
  echo -ne \\\x05\\\x01\\\x09\\\x02\\\0x05\\\x01\\\x09\\\x02\\\xa1\\\ \... > /sys/kernel/config/usb_gadget/my_gadget/functions/hid.test0//report_desc   *#这里省略了部分描述符*
  echo 4 > /sys/kernel/config/usb_gadget/my_gadget/functions/hid.test0/report_length
  echo 0 > /sys/kernel/config/usb_gadget/my_gadget/functions/hid.test0//subclass
  ln -s /sys/kernel/config/usb_gadget/my_gadget/functions/hid.test0/  /sys/kernel/config/usb_gadget/my_gadget/configs/c.1/f1   *#这一步很重要，即将上面创建的function 加入到某个congfig中,这样在host端枚举时才会认到这个function*


6.  使能gadget

> ls /sys/class/udc/| awk '{print $1}'
>
> ffd00000.dwc3
>
> ehco  ffd00000.dwc3  >   /sys/kernel/config/usb_gadget/my_gadget/UDC

这样如果USB线缆已连接主机，主机就会开始枚举。

hid设备最终的configfs目录结构

>config

>|----usb_gaget
>
>​				|----UDC
>
>​				|----bMaxPacketSize0
>
>​				|----bcdDevice
>
>​				|----bcdUSB
>
>​				|----bDeviceClass
>
>​				|----bDeviceProtocol
>
>​				|----bDeviceSubClass
>
>​				|----idProduct
>
>​				|----idVendor
>
>​				|----*configs*
>
>​						|----*c.1*
>
>​								|----*strings*
>
>​										|----*0x409*
>
>​												|----configuration
>
>​								|----MaxPower
>
>​								|----bmAttributes
>
>​								|----f1 链接文件--->hid.test0
>
>​				|----*functions*
>
>​						|----*hid.test0*
>
>​								|----protocol
>
>​								|----report_desc
>
>​								|----report_length
>
>​								|----subclass
>
>​				|----*strings*
>
>​						|----*0x409*
>
>​									|----manufacturer
>
>​									|----product
>
>​									|----serialnumber



参考/kernel/Documentation/usb/gadget_configfs.txt

