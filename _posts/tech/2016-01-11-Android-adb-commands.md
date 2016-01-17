---
layout: post
title: ADB命令大全
category: 技术
tags: ADB
description: ADB命令大全
---


查看ADB帮助`adb help`
### **设备连接**
1. 等待设备连接，在模拟器/设备连接之前把命令暂存在adb的命令器中

        adb wait-for-device 
2. 开启root权限，重启adb

        adb root 
1. 查看已连接的设备

        adb devices
7. 终止adb服务进程

        adb kill-server
8. 重启adb服务进程

        adb start-server
3. 停止监控


        Ctrl-C
### **设备管理**
3. 重启设备

        adb reboot
4. 重启到刷机模式，bootloader
    
        adb reboot bootloader
5. 重启到恢复模式，recovery
    
        adb reboot recovery
###**发送命令到设备**

        adb [-d|-e|-s <serialNumber>] <command>
        -d 发送命令给usb连接的设备
        -e 发送命令到模拟器设备
        -s <serialNumber> 发送命令到指定设备
### **APP管理**
1. 获取apk的packagename 和 classname

        aapt d badging <apkfile>
4. 安装APK

        adb install <apk>
        adb install test.apk
5. 重新安装apk，但保存数据和缓存文件

        adb install -r <apk>
        apk install -r test.apk
6. 安装apk到sd卡

        adb install -s <apk>
        adb install -s test.apk
5. 卸载apk

        adb uninstall <package>
        adb uninstall com.bob.test
6.  卸载apk，但保存数据和缓存文件     

        adb uninstall -k <package>
        adb uninstall -k com.bob.test
4. 启动应用(**Meizu Note2测试  -n  失败**)

        adb shell am start -n <package_name>/.<activity_class_name>
1. 杀死Activity

        adb shell am force-stop <package name>        
### **查看内存占用**        
4. 查看设备cpu和内存占用情况

        adb shell top
5. 刷新一次内存信息后返回,`-n`：刷新次数

        adb shell top -n 1
3. 查看占用cpu前6的进程,`-m`：前几个

        adb shell top -m 6
4. 查看线程而不是进程,`-t`：线程

        adb shell top -t
        adb shell top -t -m 6
1. 查询各进程内存使用情况(**Meizu Note2测试 失败**)

        adb shell procrank
1. 杀死一个进程

        adb shell kill <pid>
        就算手机已经root了，手机上的adbd server也拿不到root权限，命令不起作用。
        adb root也拿不到权限。
        可能的解决方案（未尝试）：安装BusyBox后：
        adb shell "su -c 'kill $(pidof <package name>)'"
        或者
        adb shell "su -c 'kill <pid>'"
2. 查看进程列表

        adb shell ps
2. 查看指定进程状态

        adb shell ps -x <pid>
2. 查看后台service列表

        adb shell service list
### **Log相关**
6. 查看log
    
        adb logcat
        adb logcat 标签:级别 *:S
        adb logcat ActivityManager:I
5. 清除log缓存

        adb logcat -c
5. 查看bug报告

        adb bugreport
3. monkey测试,500次操作

        adb shell monkey -v -p <package name> 500        
### **文件操作相关**
2. 挂载为可读写

        adb remount
1. 查看所有存储设备

        adb shell ls mnt
2. 本地推送文件到设备
    
        adb push <local file> <remote dir>
2. 从设备取出文件到本地

        adb pull <remote file> <local dir>
2. 列出目录下的文件和文件夹

        adb shell ls
3. 进入文件夹
    
        adb shell cd <dir>
3. 重命名文件

        adb shell rename path/oldname path/newname
2. 删除文件

        adb shell rm /sdcard/test.apk
3. 删除文件夹及其目录下所有文件

        adb shell rm -r <dir>
3. 移动文件

        adb shell mv oldpath/file newpath/file
4. 设置文件权限

        adb shell chmod xxx <file>
        adb shell chmod 777 /sdcard/test.apk
4. 新建文件夹

        adb shell mkdir <dir>
        adb shell mkdir path/<dir>
### **查看状态信息**
3. 查看文件内容

        adb shell cat <file>
9. 获取机器MAC地址(**Meizu Note2测试失败**)

        adb shell cat /sys/class/net/wlan0/address 
1. 获取CPU序列号

        adb shell cat /proc/cpuinfo
2. 获取当前内存占用
        
        adb shell cat /proc/meminfo
3. 查看IO内存分区
   
        adb shell cat /proc/iomem
4. 查看wifi密码(**Meizu Note2测试失败**)

        adb shell cat /data/misc/wifi/*.conf
2. 获取设备信息
        
        adb shell cat /system/build.prop
2. 获取序列号

        adb get-serialno
### **dumpsys**
    adb shell dumpsys   //获取系统状态
    adb shell dumpsys | grep "DUMP OF SERVICE"  //获取与关键字相关的状态
    adb shell dumpsys activity  //获取所有关于四大组件的信息（包括布局）,这里及之后的Activity不是指界面Activity，而是泛指Android里的四大组件。
    adb shell dumpsys activity all  //获取当前所有存在的Activity的信息
    adb shell dumsys activity activity all  //获取当前位于前台的Activity的所有信息
    adb shell dumpsys activity top  //获取前台的acitity
    adb shell dumpsys activity activities //获取所有的activity栈信息
    adb shell dumpsys activity service 关键字 //获取与关键字匹配的service
    adb shell dumpsys activity service all
    //获取所有的service   
    adb shell dumpsys activity service service //获取所有与service相关的service组件信息
    adb shell dumsys activity provider 关键字   //获取与关键字匹配的ContentProvider
    adb shell dumsys activity provider all //获取所有的ContentProvider
    

参考：
[ADB官方帮助文档][1]


  [1]: http://developer.android.com/tools/help/adb.html
