##  windows 10 开启 WSL
`WSL (Windows Subsystem for Linux)`: 适用于 Linux 的 Windows 子系统，起初是微软为了能在 Windows 10 Mobile 即 WP 中运行安卓应用而开发的，后来作为 feature 加到 Win10 系统中。这项功能使得 win10 一度成为大众口中的“最好的Linux发行版”:)  
我的win10 1709 中可以直接启用 WSL，然后再去应用商店下载安装一个发行版的应用即可。我在应用商店直接下载了 Ubuntu 18.04  
这个东西其实很鸡肋，完全可以用虚拟机或者跑个 Docker for Windows 来代替。但是我觉得 WSL 已经很完善很小白化了，开启 WSL 的成本远小于前两者。  
## WSL 有什么用
1. 用 WSL 可以免于装 Linux 双系统或者虚拟机来运行构建一些程序。比如 `C/C++` 的构建编译环境，Linux 绝对优于 windows。
2. 还可以很方便的用来学习实践 Linux 的一些基本操作，虽然它不是真正的 Linux 系统，也不能运行所有的 Linux软件。
3. 尝鲜。
## 通过 ssh 连接到 WSL
这个才是我想记录的。WSL 本来有自己的 terminal，比如我装的 Ubuntu，它就长这样![WSL-Ubuntu-terminal](https://fedt-blog.b0.upaiyun.com/uploads/1528992380000.png)  
这其实就是 又难看又难用的`powershell`!当然拒绝用这么难用的东西来玩 WSL 啦。而且，只有一直运行着这个窗口，WSL 才能工作，这么丑还不能关了它，简直了！  
所以我希望像管理服务器一样来通过 SSH 来连接 WSL 来就行一系列的操作，并且把 WSL 作为我 win10 的一个服务，开机启动并能直接 SSH 连上。  

想要能够 SSH 连接，则 WSL 上必须有 `ssh server`，这个安装了 Ubuntu 自带的，但是都说自带的不好用，那么为了避免踩坑，在 WSL 中重装下ssh server:  
```bash
// 卸载
sudo apt-get remove openssh-server
// 安装
sudo apt-get install openssh-server
```
如果 root 用户，则不需要 `sudo` 命令提权。  
还要简单改下 `ssh` 配置：
```bash
# 编辑配置文件
vim /etc/ssh/sshd_config
    
  Port 2222  # 默认的是22，但是windows有自己的ssh服务用的也是22端口，修改一下
  PasswordAuthentication yes # 允许用户密码方式连接
  PermitRootLogin yes # 允许 root 用户连接

# 重启ssh服务
sudo service ssh --full-restart
```
通过 Google 搜到基本上只有改端口号跟运行用户名密码这两项配置，但是作为小白我喜欢用 root 用户操作，所以 `PermitRootLogin` 必须开启，因为这个搞了好半天。  

其实，如果是服务器，一般都会改默认端口，不允许用户名密码方式连接，不允许 root 用户连接的。这些是基本的安全措施。  
这时候就应该用密钥的方式来连接：  
首先用 xshell 生成公钥(Pubic Key)与私钥(Private Key)，导出公钥并将其放到 WSL：  
```bash
# 公钥需要被粘贴在 /root/.ssh/authorized_keys
mkdir -p /root/.ssh 
# 假定公钥零时存放在 /root/id_rsa_1024_20140305.pub
mv /root/id_rsa_1024_20140305.pub /root/.ssh/authorized_keys 
# 给读写权限
chmod 600 /root/.ssh/authorized_keys
```  
然后修改 ssh 相关配置：  
```bash
vim /etc/ssh/sshd_config
  PubkeyAuthentication yes    #启用公告密钥配对认证方式 
  AuthorizedKeysFile .ssh/authorized_keys     #设定PublicKey文件路径
  RSAAuthentication yes  #允许RSA密钥
  PasswordAuthentication no #禁止密码验证登录,如果启用的话,RSA认证登录就没有意义了

# 重启ssh服务
sudo service ssh --full-restart
# 或者这样重启
sudo /etc/init.d/sshd restart 
```
至此就可以使用密钥的方式来通过 ssh 连接到 WSL.  

## 自动启动 WSL 并运行 ssh server
现在需要做的是 win10 系统启动后，自动后台运行（看不到那个很丑的 terminal）WSL，并自动启动 WSL 的 ssh server。我[搜到了两段脚本](https://gist.github.com/dentechy/de2be62b55cfd234681921d5a8b6be11)来做这个事.  
首先自动后台启动 WSL :  
```bash
vi sshd.bat
  # 我用的 C:\Windows\System32\bash.exe -c "sudo /etc/init.d/ssh start -D" 其实是一样的，只是下面这个可能会报错...
  C:\Windows\System32\bash.exe -c "sudo /usr/sbin/sshd -D"

mv ssh.bat /mnt/c/Users/YourUserName/Documents
```
其实就是一段 bat 脚本，启动 bash(即 WSL)并 启动 ssh server，这脚本存放的实际位置是 `C:\Users\YourUserName\Documents` (这个存放位置其实也无所谓，只要跟后面那个放一起就行)。注意 YourUserName 得换成真实的 win10 用户名。

然后需要一段 `VBScript` 脚本来执行这个 bat:  
```bash
vi sshd.vbs
  Set WinScriptHost = CreateObject("WScript.Shell")
  WinScriptHost.Run Chr(34) & "C:\Users\YourUserName\Documents\sshd.bat" & Chr(34), 0
  Set WinScriptHost = Nothing

mv sshd.vbs /mnt/c/Users/YourUserName/Documents
```
vbs 代码看不懂，看意思就是用来执行上面那个 bat 脚本的。  
windows 系统可以在开始菜单文件夹中添加脚本达到开机执行的目的，cmd 指令是 `shell:startup`，打开开始菜单文件夹后，把 `sshd.vbs` 复制进去，这样每次 win10 启动就会带起 WSL 了。  

到这儿其实已经差不多了，但是如果 WSL 默认用户不是 root,则上面的 `sudo /usr/sbin/sshd -D` 因为要输入用户密码而不能顺利执行，解决办法是在 WSL 中配置这条命令不需要输入密码来绕过，在 WSL 中：
```bash
sudo visudo # 等同于 vim /etc/sudoers
# 特别提示： 这个 sudo visudo 命令是使用 GNU nano 来编辑 /etc/sudoers 的，这东西保存并结束输入的快捷键是 Enter ... 这个还蛮坑的，半天找不到怎么保存退出
```
在打开的配置文件中加上 `%sudo ALL=NOPASSWD: /usr/sbin/sshd` 来绕过 sudo 提权时的密码输入，这样 bat 脚本中的 `sudo /usr/sbin/sshd -D` 可以顺利执行。


完成上述操作，即可在 WIN10 中畅玩 WSL 了。通过这番折腾，简单熟悉了下 ssh server 的相关配置，安全性较低的 ssh 连接对于 Linux 服务器简直是灾难，很多的云服务器入侵就是扫描 22 端口直接 ssh 连到服务器的，所以这些基本操作还是要必要掌握下。

## 参考链接
- [Secure Shell](https://zh.wikipedia.org/wiki/Secure_Shell)
- [**How to automatically start ssh server on boot on Windows Subsystem for Linux**](https://gist.github.com/dentechy/de2be62b55cfd234681921d5a8b6be11)
- [使用xshell登录ubuntu on windows(wsl)](https://www.jianshu.com/p/039411d2c1f6)
- [利用xshell密钥管理服务器远程登录](http://blog.51cto.com/zengweidao/1437979)
- [sudoers的深入剖析与用户权限控制](https://segmentfault.com/a/1190000007394449)

