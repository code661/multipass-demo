> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [nextfe.com](https://nextfe.com/multipass/)

作者：[LeanCloud weakish](https://leancloud.cn/)

- [安装](#安装)
- [上手](#上手)
- [定制](#定制)
- [更多](#更多)
- [结语](#结语)


容器技术可以保证环境一致性，简化项目配置、部署流程，因此很受广大开发者青睐。 如果你打算尝试或者已经尝试基于容器简化本地项目环境配置，但又嫌弃 [docker](https://nextfe.com/docker-for-frontend-developers/) 用起来还是不够直截了当，那么可以试下 [multipass](https://multipass.run/)。

安装
--

macOS（支持 Sierra 以上版本）可以直接通过 Homebrew 安装：

```bash
brew cask install multipass
```

Windows 用户可以到[这里](https://github.com/canonical/multipass/releases/download/v1.1.0/multipass-1.1.0%2Bwin-win64.exe)下载安装（只支持 Windows 10，如果是 Windows 家庭版或者 v1803 之前的 Windows 10 专业版 / 企业版，还需要另外安装 VirtualBox）。

运行一下 `multipass version` 命令确认安装成功，顺便查看一下版本：

```
multipass  1.1.0+mac
multipassd 1.1.0+mac
```

可以看到当前版本是这个月刚发布的 1.1.0。 对很多用户来说，这个版本最大的更新是支持代理。 从 1.1.0 起，multipass 像很多命令行工具一样，会遵循 `http_proxy` 环境变量中指定的代理。 因为 multipass 创建容器时可能需要从网络下载镜像，而很多地方的网络连通性不尽如人意，因此支持代理能够大大改善使用体验。

上手
--

先来创建一个容器：

```
$ multipass launch --name react
Launched: react
```

初次创建时需要下载镜像，网络畅通的情况下，稍等片刻即可。

容器创建后 multipass 会马上启动它，这样创建好容器后我们就可以直接使用了：

```
$ multipass exec react -- lsb_release -d
Description:	Ubuntu 18.04.4 LTS
```

`lsb_release` 会打印 Linux 发行版的信息。 之前我们创建容器的时候并没有指定使用什么样的镜像，上面命令的输出表明，multipass 默认会使用当前 LTS 版本的 Ubuntu。

除了直接在容器上运行（`exec`）命令外，还可以通过 `shell` 命令「进入」容器：

我们进入了一个完整的 Linux 环境，可以进行各种操作。 例如，假设我们看到了[一篇介绍 React Hooks 的教程](https://nextfe.com/react-hooks/)，打算体验一下教程的示例项目：

```
multipass shell react
```

哎呀，系统告诉我们 `npm` 没有安装，并建议通过 `apt` 安装。

```
git clone https://github.com/hjiang/react-hook-demo.git
cd react-hook-demo
npm install
```

不过，当前 LTS 版本的 Ubuntu 仓库里的 Node.js 比较老旧，我们转而安装 LTS 版本的 Node.js（12）：

```
The program 'npm' is currently not installed. You can install it by typing:
sudo apt install npm
```

项目跑起来了，太棒了：

```
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs
npm install
npm start
```

这里 `192.168.64.5` 是 Multipass 分配给 react 这个容器的 IP，所以我们可以直接在宿主机上打开浏览器访问 `http://192.168.64.5:3000/` 查看效果。

定制
--

接下来我们尝试容器化手头正在开发的一个 Node.js 项目。 和之前不同，我们将对容器进行一些定制，这样用起来更方便。

首先，我们的 Node.js 项目将部署到云平台，所以我们希望容器的规格尽可能和云平台上的生产环境一致。 其次，我们之前手动安装了 Node.js，这次我们希望自动化这一安装过程。

因此，我们使用以下命令创建容器：

```
Compiled successfully!

You can now view leancloud-react-hook-tutorial in the browser.

  Local:            http://localhost:3000/
  On Your Network:  http://192.168.64.5:3000/

Note that the development build is not optimized.
To create a production build, use npm run build.
```

我们通过命令行参数指定了容器的磁盘和内存大小，并且显式指定使用 Ubuntu 18.04。 容器创建成功后，通过 `multipass info` 可以查看容器的基本信息：

```
multipass launch --name lean --disk 2G --mem 256M --cloud-init lean.yaml 18.04
```

可以看到，之前创建的 react 容器，multipass 默认分配了 5G 硬盘和 1G 内存。 而 lean 容器则按照我们的要求分配了 2G 硬盘和 256M 内存（这是我们计划使用的云平台 [LeanCloud 云引擎](https://www.leancloud.cn/engine/) 免费版体验实例的规格）。 另外，基本信息中没有 CPU 核心的信息，multipass 默认会给容器分配 1 个 CPU 核心。

至于 `lean.yaml` 则是容器的初始化配置文件，内容如下：

```
$ multipass info --all
Name:           lean
State:          Running
IPv4:           192.168.64.2
Release:        Ubuntu 18.04.4 LTS
Image hash:     fe3030939822 (Ubuntu 18.04 LTS)
Load:           0.11 0.30 0.16
Disk usage:     1.3G out of 2.0G
Memory usage:   71.4M out of 229.7M

Name:           react
State:          Running
IPv4:           192.168.64.5
Release:        Ubuntu 18.04.4 LTS
Image hash:     fe3030939822 (Ubuntu 18.04 LTS)
Load:           0.00 0.00 0.00
Disk usage:     1.7G out of 4.7G
Memory usage:   112.1M out of 985.7M
```

`runcmd` 可以指定容器 **首次启动** 时运行的命令，这里我们复制了之前安装 Node.js 的命令，还加上了安装 `lean-cli` 的命令（我们通过 `lean-cli` 将代码部署到云平台）。

容器初始化配置文件遵循 [cloud-init](https://cloudinit.readthedocs.io/en/latest/topics/examples.html) 标准，可以通过 yaml 文件进行用户、文件、软件仓库、 DNS 解析、SSH 密钥、puppet、chef 等各种初始化配置。

我们只打算在容器中测试、部署项目，并不打算 `multipass shell` 到容器内使用 vim 或 emacs 开发项目。 所以，我们直接挂载宿主机上的一个目录：

```
#cloud-config

runcmd:
  - curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
  - sudo apt-get install -y nodejs
  - wget https://releases.leanapp.cn/leancloud/lean-cli/releases/download/v0.21.0/lean-cli-x64.deb
  - sudo dpkg -i lean-cli-x64.deb
```

> `demo` 是我们的 Node.js 项目目录，如果读者想要测试，可以用下面这个模板项目：
>
> ```
> multipass mount demo lean:/home/ubuntu/demo
> ```
>
> 同时去 [LeanCloud](https://leancloud.app/) 注册账号、创建应用，方便体验下面的部署操作。

挂载完成后，我们就可以在宿主机上使用趁手的 [IDE](https://nextfe.com/jetbrains-ide-shortcuts/)、[编辑器](https://nextfe.com/vscode-extensions/)开发项目，之后 `multipass shell lean` 到容器内测试：

```
git clone https://github.com/leancloud/node-js-getting-started demo
```

屏幕会输出：

```
cd demo
lean login # 使用之前注册的 LeanCloud 账号登录
lean switch # 选择之前创建的应用
npm install # 安装项目依赖
lean up # 本地（容器内）调试
```

之前通过 `multipass info`，我们知道 lean 容器的 IP 是 `http://192.168.64.2`，所以在宿主机上访问 `http://192.168.64.2:3000/` 即可查看效果。 如果效果符合预期，我们可以在容器内运行 `lean deploy --prod 1` 部署项目。

更多
--

运行 `multipass list` 可以列出所有的容器：

```
Node app is running on port: 3000
```

如果希望节约资源，我们可以停止暂时用不到的容器，比如之前创建的 react：

之后我们可以运行 `multipass start react` 重新运行容器。 如果以后不再使用，那么也可以干脆删除：

```
Name                    State             IPv4             Image
lean                    Running           192.168.64.2     Ubuntu 18.04 LTS
react                   Running           192.168.64.5     Ubuntu 18.04 LTS
```

最后，很多时候，我们只是想要在 macOS 或 Windows 上起一个 Linux 环境，然后进行一些操作，multipass 应付这一使用场景最是得心应手：

是的，你没看错，只需一条命令，你就可以进入一个与宿主机隔离的 Linux 容器！ multipass 会自动创建并运行一个名为 Primary 的容器（如果还没有创建或运行的话），这个容器也会自动挂载宿主机的 Home 目录，就是这么省心省力。

结语
--

Multipass 使用起来十分简洁直观。 它是由 Canonical （Ubuntu 背后的公司）推出的，因此使用的镜像由 Canonical 负责更新，包含最近的安全更新，以及专门为各个平台的虚拟化方案（Windows 的 Hyper-V、macOS 的 HyperKit、Linux 的 KVM）优化的内核。 不过也因为同样的原因，目前支持的镜像也只限于 Ubuntu。

题图来自 [multipass 官网](https://multipass.run/)
