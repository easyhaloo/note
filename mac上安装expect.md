### 必备软件

1. [tcl](<http://www.tcl.tk/software/tcltk/downloadnow84.tml>)

2. [expect](<http://www.linuxfromscratch.org/blfs/view/svn/general/expect.html>)

   下载软件的地址建议放在`/usr/local`目录下。



#### 1. 配置tcl

-  解压&&编译

```shell
cd tcl8.4.20
cd unix
sudo ./configure --prefix=/usr/local/tcl --enable=shared
sudo make 
sudo make install
sudo cp ./tclUnixPort.h ../generic/
```



#### 2.配置expect

- 解压&&配置

```shell
sudo tar -zxvf expect.5.45.tar.gz
cd expect5.45
sudo ./configure --prefix=/usr/local/expect --with-tcl=/usr/local/tcl/lib --with-tclinclude=/usr/local/tcl8.4.20/generic
sudo make
sudo make install
```



#### 3.配置远程免密登陆

- 创建shell脚本

```shell
sudo vim login.sh

#下面的贴进去
#脚本段 start
#! /usr/bin/expect

spawn ssh root@102.12.12.12
expect "*Password:"
send "password\n"
expect "*#"
interact
#脚本段 over

sudo chmod +x login.sh

#执行脚本
./login.sh 
# 登陆需要时间，这个受网络环境影响
```





