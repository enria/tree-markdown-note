1. ##### 给root用户设置密码

  ```shell
sudo passwd root
  ```

会提示输入unix的新密码，这就是root的密码



2. ##### 测试root用户登录

` sudo su`  或者 `su root`



3. ##### 修改配置文件，开启root账号界面登录

```shell
cd /usr/share/lightdm/lightdm.conf.d
gedit 50-unity-greeter.conf
```

在下面添加如下配置：

```shell
user-session=ubuntu
greeter-show-manual-login=true

all-guest=false
```



4. ##### 重启系统，使用root账号登录

```shell
reboot
```

会出现一个错误提示，点击确定进入系统



5. ##### 编辑/etc/lightdm/lightdm.conf

```shell
sudo gedit  /etc/lightdm/lightdm.conf
```

```shell
[Seat:*]
autologin-guest=false
autologin-user=root
autologin-user-timeout=0
greeter-session=lightdm-gtk-greeter 
```



6. ##### 修改配置文件

```shell
 gedit /root/.profile
```

把`mesg n`修改为：`tty -s && mesg n`

保存退出，重启系统

```shell
 reboot
```



7. ##### 安装open-ssh模块开启远程登录

```shell
 apt-get install openssh-server
```

查看是否开启：`ps -e |grep ssh`



8. ##### 开启root远程登录

```shell
 gedit /etc/ssh/sshd_config
```


找到并用注释掉这行：`PermitRootLogin prohibit-password`

新建一行 添加：`PermitRootLogin yes`

重启服务

```shell
 service ssh restart
```

