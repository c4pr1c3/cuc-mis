# 实验

## 无线路由器/无线接入点（AP）配置

以下实验，默认配置的是AP，除非特别说明时会强调该实验内容需要无线路由器支持。

* 重置和恢复AP到出厂默认设置状态
* 设置AP的管理员用户名和密码
* 设置SSID广播和非广播模式
* 配置不同的加密方式
* 设置AP管理密码
* 配置无线路由器使用自定义的DNS解析服务器 
* 配置DHCP和禁用DHCP
* 开启路由器/AP的日志记录功能（对指定事件记录）
* 配置AP隔离(WLAN划分)功能
* 设置MAC地址过滤规则（ACL地址过滤器）
* 查看WPS功能的支持情况
* 查看AP/无线路由器支持哪些工作模式

## 使用手机连接不同配置状态下的AP对比实验

* 如果手机无法分配到IP地址但又想联网该如何解决？

## 使用路由器/AP的配置导出备份功能，尝试解码导出的配置文件

下图以TP-LINK TL-WR720N为例：

![](attach/chap0x01/tp-link tl-wr720n backup and restore.png)

提示：

* 在Google上搜索：路由器品牌名 backup decoder，例如：tp link router backup decoder

![](attach/chap0x01/Google tp link router backup decoder.png)

打开[Zibri's Blog: TP-LINK Configuration file encrypt and decrypt](http://www.zibri.org/2015/10/tp-link-configuration-file-encrypt-and-decrypt.html)，上传上一步备份导出的路由器配置文件会自动得到一份解码之后的明文路由器配置文件。

* 检查一下明文配置文件内容，看看你的路由器设置是不是在这份配置文件里都可以找到？
* 用浏览器的调试工具分析一下这个【备份配置文件】功能的网络通信过程，有没有安全风险？

## 复习VirtualBox的配置与使用

* 虚拟机镜像列表
* 设置虚拟机和宿主机的文件共享，实现宿主机和虚拟机的双向文件共享
* 虚拟机镜像备份和还原的方法
* 熟悉虚拟机基本网络配置，了解不同联网模式的典型应用场景

