# Linux服务器安装Python

## 安装Python

### 1.安装依赖环境

```# yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel```

### 2.浏览器打开 [ftp/python](https://www.python.org/ftp/python/) 查看最新的Python版本，标记为3.A.B

```# wget https://www.python.org/ftp/python/3.A.B/Python-3.A.B.tgz```

本地下载tgz文件，通过FileZilla上传到服务器

### 3.创建Python3的目录，python将会安装到这个目录下

```# mkdir /usr/local/python3```

### 4.解压下载文件并切换目录

```# tar -zxvf Python-3.A.B.tgz```

```# cd Python-3.A.B```

### 5.配置安装的位置并安装

```# ./configure --prefix=/usr/local/python3```

```# make && make install```

### 6.创建Python3的软链接，这样就可以调用直接调用python命令

```# ln -s /usr/local/python3/bin/python3 /usr/bin/python3```

### 7.创建Pip3的软链接

```# ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3```

### 8.测试命令 python3 和 pip3

```# python3```

```# pip3```

## 离线安装python包

### 1.本地通过pip下载包

```# pip download package -d D:\temp```

可以使用国内的镜像：`# pip install package -i http://pypi.douban.com/simple --trusted-host pypi.douban.com`

### 2.将pip下载的whl文件上传到服务器

### 3.在服务器上使用pip进行安装

```# pip install package.whl```

### 4.打开python，再import包，如果没有出错就安装成功了