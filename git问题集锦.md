## 主要记录下使用Git出现的问题



#### 1. Clone||Pull远端仓库失败或中断

- 设置`git`的`http.postBuffer`属性

```shell
git config --global http.postBuffer 524288000
```

- 更改`clone`方式为`ssh`
- 终极大招。。。手动下载

---

#### 2. clone速度慢

> 由于git的地址在国外，一般都会限速,可以选择配置hosts地址映射来加速访问。

- 配置映射

  一般我们需要映射两个地址：

  [github.global.ssl.fastly.net](github.global.ssl.fastly.net)

  [github.com](github.com)

  可以选择`ping`命令，或者通过[站长工具](<http://tool.chinaz.com/dns/?type=1&host=github.global.ssl.fastly.net&ip=>)查询对应的IP地址，尽量选择`TTL`低的节点，这样可以大幅度降低延迟。

- 配置hosts

  ```shell
  sudo vim /etc/hosts
  #配置这个主机地址
  xxx.xx.xx.xx github.global.ssl.fastly.net
  xx.xx.xx.xx github.com
  
  # 如果是window 请寻找/etc/hosts目录
  ```

- 刷新DNS

  ```shell
  # linux
  sudo /etc/init.d/networking restart
  ## 或者
  sudo service network restart
  
  ## window
  ipconfig /flushdns
  
  ## mac   下面几条命令任选其一。由于mac版本不同造成的
  lookupd -flushcache
  type dscacheutil -flushcache
  sudo killall -HUP mDNSResponder
  ```

---

#### 3. Git仓库的迁移

- 从原仓库克隆裸版本库,不包含代码

  ```shell
  git clone --bare git://github.com/username/project.git
  ```

- 推送版本库到你需要迁移到的仓库中

  ```shell
  # project.git 克隆的裸版本库信息
  cd project.git
  git push --mirror git@github.com/username/newproject.git
  ```

- 现在你可以克隆你的新仓库了

  ```shell
  git clone git@github.com/username/newproject.git
  ```



#### 4. Git回滚到制定版

- 查找`log`

  ```shell
  git log
  ```

  查找`log`可以看到`commit`的详细信息。`commit`后面跟着到一串字符串就是`commit id`

- 回滚版本

  ```shell
  git reset --hard xxxxx
  ```

  xxx 代表`commit id`

- 

















