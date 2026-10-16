    ---
    layout: post
    title: "ThinkPad T480-安装Linux指纹驱动"
    date: 2026-10-13
    categories: jekyll
    tags: ThinkPad
    ---

# ThinkPad T480 安装指纹驱动经验分享

## ⚠️前言

- 本文不适合纯新手，不要盲目复制粘贴运行命令或执行操作！执行操作前请明确代码含义与后果！
- 驱动程序其实采用**逆向工程**，亲测**效果不佳**，只能达到“可用”的水平，折腾、研究、学习作用大于实用性;
- 本文**不是专业教程文档**，仅仅是我自己的实操**经验分享**，大部分内容均来自网络，且没考虑到更多运行环境，还请各位结合具体情况具体分析;
- 本人对Linux也是一知半解，再加上时间拖得太长了，很多细节记不清了，如有错误欢迎交流指正。
- 本文的**卸载方法没经过实测**，建议设置**系统快照**以免污染系统环境
- 进行危险操作前别忘了**按需备份**

## 本文相关测试环境

### 硬件环境

| 类别       | 名称代号                                      | 厂商             |
| -------- | ----------------------------------------- | -------------- |
| 整机       | ThinkPad T480 20L5                        | Lenovo         |
| **指纹模块** | **Synaptics WBDI - SGX**                  | Synaptics, Inc |
| 主板产品     | 20L5A070CD                                | LENOVO         |
| 主板型号     | SDK0L77769 WIN                            | LENOVO         |
| 芯片组      | Intel Sunrise Point-LP, Intel Kaby Lake-R | Intel          |

### 软件环境

| 类别           | 版本                                  |                           |
| ------------ | ----------------------------------- | ------------------------- |
| **Linux 内核** | **6.14 LTS**                        |                           |
| **传感器主线**    | **06cb:009a**                       |                           |
| BIOS 模式      | UEFI                                |                           |
| BIOS         | LENOVO N24ET56W (1.31 ), 2020/2/19  | Phoenix Technologies Ltd. |
| 发行版          | Debian GNU/Linux 13 (trixie) x86_64 |                           |
| 桌面环境         | Xfce 4                              |                           |

## 内核 ⇆ 传感器 测试

### 设置 BIOS

- BIOS 关闭 **Secure Boot**（否则固件加载失败，驱动无法启动）

- BIOS 开启 Fingerprint Reader，关闭 Windows 专属模式

### 宏内核 驱动测试

```bash
lsusb | grep 06cb
```

此时应当完返回：`Bus 001 Device 006: ID 06cb:009a Synaptics, Inc. Metallica MIS Touch Fingerprint Reader` 类似的设备型号。这就说明内核已正确识别到硬件，但还无法进行交互。

如果没有，可以尝试安装 `fprintd` 以及相关包（现代包管理器大概率可以自动补全依赖关系），

```bash
sudo apt update
```

```bash
sudo apt install fprintd
```

后 注销 或 重启。再次尝试

如果依然无法识别请检查：

- BIOS 是否禁用 指纹传感器硬件
- 操作系统内是否禁用指纹传感器硬件

还可以尝试更换内核版本 或 尝试其他发行版。

## 核心组件

### fprintd

提供 CLI外壳，用于 `fprintd-enroll` 等基础命令的支持

#### 安装

```bash
sudo apt install fprintd libpam-fprintd
```

#### 禁用服务

禁止系统自动启动官方 fprintd 服务（避免之后设备抢占冲突）

```bash
sudo systemctl mask fprintd
```

停用原版 fprintd 守护进程（考虑到终端兼容性，最好逐条运行）

```bash
sudo systemctl stop fprintd
sudo systemctl disable fprintd
```

再次确认

```bash
systemctl status fprintd
```

应当返回 `disabled / inactive`

#### 手动提取 CLI外壳程序与相关组件

##### 提取 CLI外壳程序

找到这些程序，并复制一份到其他目录备用。

```bash
which fprintd-enroll
```

一般会输出：`/usr/bin/fprintd-enroll` 这就是程序的目录

这是⬆️核心外壳程序，负责指纹录入。如果需要完整体验，还需要找到⬇️

| 程序             | 用途       |
|:--------------:|:--------:|
| fprintd-verify | 指纹验证测试   |
| fprintd-list   | 列出所有指纹   |
| fprintd-delete | 删除指纹     |
| fprintd-verify | 验证已录入的指纹 |

如果找不到也可以手动下载 fprintd 的 deb包，解包提取。

##### 提取 动态链接库

`pam_fprintd.so`是 `libpam-fprintd`带的动态链接库，如果正确安装了`libpam-fprintd`那么它大概率在`/usr/lib/security/pam_fprintd.so`，如果不在也可以使用检索工具搜索。复制备用。

`libpam-fprintd`是`fprintd`的依赖捆绑包，btw

### python-validity （传感器核心驱动）

> 我查资料的时候发现 AUR 和 PPA 源中似乎都有现成编译好的包和安装脚本，但由于本文的“教具”是Debian，所以暂不演示

#### 1.安装依赖

```bash
sudo apt install git build-essential python3 python3-pip python3-dev libusb-1.0-0-dev libssl-dev fprintd libpam-fprintd
```

#### 2.编译安装

```bash
git clone https://github.com/uunicorn/python-validity.git
cd python-validity
sudo pip3 install .
```

##### Debian13（Python3.13）PEP 668 外部托管环境保护

Debian13（Python3.13）启用了 PEP 668 外部托管环境保护，禁止直接用 `sudo pip3 install .` 往系统 Python 里安装包，所以报错 `externally-managed-environment`。

> `sudo pip3 install .`

表现报错：

```
error: externally-managed-environment
× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
python3-xyz, where xyz is the package you are trying to
install.
If you wish to install a non-Debian-packaged Python package,
create a virtual environment using python3 -m venv path/to/venv.
Then use path/to/venv/bin/python and path/to/venv/bin/pip. Make
sure you have python3-full installed.
If you wish to install a non-Debian packaged Python application,
it may be easiest to use pipx install xyz, which will manage a
virtual environment for you. Make sure you have pipx installed.
See /usr/share/doc/python3.13/README.venv for more information.
note: If you believe this is a mistake, please contact your Python installation or OS distribution provider. You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.
```

解决方式（**三选一**）：

###### 方案A 虚拟环境编译

这是网上比较推荐的方案，安全、干净、可控，不污染系统Python。

1. 进入 python-validity 源码目录
   
   ```bash
   cd python-validity
   ```

2. 创建虚拟环境
   
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

3. 在 venv 内安装（此时不受PEP668限制）
   
   ```bash
   pip3 install .
   ```

4. 下载固件（venv里的python）
   
   ```bash
   python3 ./validity-sensors-firmware
   ```

5. 复制程序、固件、systemd服务文件到系统目录
   
   ```bash
   sudo cp venv/bin/validity-sensors-firmware /usr/local/bin/
   sudo cp venv/bin/python3-validity /usr/local/bin/
   sudo cp python3-validity.service /etc/systemd/system/
   sudo cp -r venv/share/python3-validity /usr/share/
   ```

###### 方案B 临时 关闭限制

> 这毕竟是 GNU/Linux 系统，你想对TA干什么都可以，只要你想，只要你有 SuperUser权限。

```bash
sudo pip3 install . --break-system-packages
```

> ⚠️ 警告：这个参数会绕过 Debian 保护机制，有可能破坏系统 Python，后续 apt 相关 Python 组件报错。仅应急测试使用。

然后继续：

```bash
 sudo python3 ./validity-sensors-firmware
```

### open-fprintd（兼容层服务）

#### 安装

###### 将源码克隆到本地

```bash
git clone https://github.com/uunicorn/open-fprintd.git
```

```bash
cd open-fprintd
```

###### 编译安装

```bash
sudo make install
```

`open-fprintd`源码安装的时候，会往系统注册 `Provides: fprintd`，此时就会提示要求REMOVE之前安装的 `fprintd`与 `libpam-fprintd`，如果没有提示，就需要手动卸载`fprintd`与`libpam-fprintd` 。

### 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now python3-validity
sudo systemctl enable --now open-fprintd-resume open-fprintd-suspend
```

#### 验证服务状态

```bash
systemctl status python3-validity
```

```bash
systemctl status open-fprintd-resume
```

查看返回是否是 “active running”

### 拼装组件

#### 解析

`fprintd`包含两类东西：

fprintd daemon（DBus 后台服务，和 open-fprintd 互斥）

客户端工具 fprintd-enroll / fprintd-list / fprintd-verify.....

我们要做的就是**把 `open-fprintd`服务 拼到 `fprintd`官方客户端上**，因为官方服务（fprintd daemon）不支持`python-validity`驱动；

`pam_fprintd.so`是向 Unix pam\-auth 添加条目与动态链接的，这样系统才能调用指纹作为验证

#### CLI外壳客户端

将提前备份的 fprintd-enroll .....等复制回原处（之前安装`open-fprintd`时这些东西应该被联动删除了）。

##### 录入测试

录入指纹

```bash
fprintd-enroll
```

此时指纹指示灯应当亮起，程序进入录入引导

查看已录入指纹

```bash
fprintd-list $USER
```

验证指纹可用性

```bash
fprintd-verify
```

至此：指纹硬件驱动、录入、验证功能完成。

如果出现问题，可以尝试**重启 dbus** （注意要在TTY模式下操作）有时候可以解决 DBus 通信异常：

```
sudo systemctl restart dbus
```

其实我更推荐注销，重新登录。或重启系统，后要再次检查`systemctl`服务状态

#### 动态链接库

将`pam_fprintd.so`复制回`/usr/lib/security/`

验证文件是否到位

```bash
ls /usr/lib/security/pam_fprintd.so
```

##### 配置指纹认证

注意做好备份

###### sudo 权限认证

使用 root权限 编辑 `/etc/pam.d/sudo`

在文件**第一行顶部**添加：

`auth sufficient pam_fprintd.so`

###### 系统登录/锁屏认证

使用 root权限 编辑 `/etc/pam.d/common-auth`

在密码验证行上方添加，参数可以根据个人喜好填写

`auth [success=2 default=ignore] pam_fprintd.so max-tries=3 timeout=10`

，而后重启登录管理器。

###### 系统授权弹窗验证

我用的是`polkit`所以

应当以root权限编辑`/etc/pam.d/polkit-1`

文件顶部添加：

`auth sufficient pam_fprintd.so`

至此基本大功告成了。

## 后续事项

- 系统内核/大版本更新后，若指纹失效，重新编译 python-validity 即可
- 最好 `hold`住`fprintd`以及相关组件，以免被其他程序再次唤起安装。我不清楚会不会和某些桌面环境自带的指纹管理器冲突，还请谨慎操作
- 装好后也不要打开Secure Boot大概率会导致失效

### 卸载

我没试过，可以自己研究研究，不过理论上讲比较繁琐但也相对简单，应该只需要一步一步反向操作就可以（毕竟都是手动、半自动安装的）。

1. 清理 PAM 配置文档
2. 清理全局 common-auth
3. 清理 polkit 授权弹窗
4. （可选）删除 pam-auth-update 自定义配置文件
5. 停止并删除 systemd 服务
6. 删除 python3-validity 与 open-fprintd
7. 解锁官方 fprintd 服务（取消 mask 锁定）

还有个“邪修”的办法，！！不要轻易尝试！！直接执行

```bash
sudo apt install fprintd
```

因为之前说过相关组件冲突，安装`fprintd`会触发 `apt`自动处理重读关系，相关组件就会被移除、替换。

最后再 `purge`掉即可。这么做可能会留下不少隐患！
