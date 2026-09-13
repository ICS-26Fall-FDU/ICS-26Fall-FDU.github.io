--- 
title: 使用 SSH
---
# 使用 SSH

SSH (Secure Shell) 是一种工作在应用层的加密网络传输协议。它的应用场景相当广泛，比如访问远程终端、传输文件、端口转发等等，Git 同样可以使用 SSH 作为网络协议来传输数据。

SSH 的架构设计可参见 [RFC 4251](https://datatracker.ietf.org/doc/html/rfc4251/)[^1]，其三层协议按从低到高的顺序分别记录在 [RFC 4253](https://datatracker.ietf.org/doc/html/rfc4253)、[RFC 4252](https://datatracker.ietf.org/doc/html/rfc4252) 和 [RFC 4254](https://datatracker.ietf.org/doc/html/rfc4254)。

[^1]: RFC 是 Request for Comments 的缩写，是由 [IETF](https://www.ietf.org/) 发布的一系列备忘录，当中包括了许多记录互联网规范、协议、过程等的标准文件。

!!! note "读一读"

    请阅读 RFC 4251 的 Introduction 部分，了解 SSH 的三层协议有哪些。

这三层协议分别是传输层协议（Transport Layer Protocol）、用户认证协议（User Authentication Protocol）和连接协议（Connection Protocol），它们分别提供如下功能：

- 传输层协议：协商对称加密密钥、验证服务器身份并提供加密。
- 用户认证协议：进行客户端身份认证。
- 连接协议：承载实际的业务并提供多路复用功能。

本文档将会帮助同学们在类 Unix 操作系统下学习 OpenSSH 的基本使用，对协议本身不会涉及太深。

### OpenSSH 简介

OpenSSH 是一个广受欢迎的 SSH 协议自由软件实现。它最初是为 OpenBSD 开发的，为了在其他类 Unix 系统上运行，OpenBSD 团队还维护了一个[移植版](https://github.com/openssh/openssh-portable)。今天的大多数 Linux 发行版和 macOS 都内置了 OpenSSH 的这一移植版。

OpenSSH 提供了一系列 SSH 客户端和服务端工具：

> The OpenSSH suite consists of the following tools:
>
> - Remote operations are done using `ssh`, `scp`, and `sftp`.
> - Key management with `ssh-add`, `ssh-keysign`, `ssh-keyscan`, and `ssh-keygen`.
> - The service side consists of `sshd`, `sftp-server`, and `ssh-agent`.
>
> -- <https://www.openssh.org>

我们在本文中只介绍课程需要的 `ssh` 和 `ssh-keygen`。对于其他工具，大家可以自行了解。

### 生成一对密钥

!!! info "你知道吗？"

    当我们发起一个 SSH 连接到其他的电脑时，我们的计算机叫作客户端，对方叫作服务器。

为了确保只有我们可以登录服务器，我们需要在 SSH 连接时向服务器证明我们的身份。其中一个证明方式是使用 **公钥认证**，这需要我们来生成一对密钥。

!!! info "公钥认证"

    公钥认证的核心机制是数字签名，它是用公钥密码算法实现的。[这篇文章](https://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)介绍了数字签名的原理。

!!! info "你知道吗？"

    SSH 提供了不止一种用户认证方法。

    !!! note "读一读"
        请阅读**用户认证协议**对应的 RFC，看看其中规定了有哪些认证方法！

    在这之后，SSH 的认证方法在 [RFC 4256](https://datatracker.ietf.org/doc/html/rfc4256)、[RFC 4462](https://datatracker.ietf.org/doc/html/rfc4462) 等标准中又有所扩展，但总的来说，公钥认证和密码认证是最常用的两种认证方法，我们这里介绍的是公钥认证。
    

OpenSSH 为我们提供了 `ssh-keygen` 来便捷地生成一对密钥。`ssh-keygen` 可以生成多种算法的密钥对，我们建议大家使用基于椭圆曲线密码学的算法，比如 Ed25519，它和 RSA 相比密钥尺寸更小，性能也相对更好。

!!! note "读一读"
    在终端中输入以下内容并按下回车：
    ```bash
    man ssh-keygen # 其中 man 是 manual 的意思
    ```
    这是 `ssh-keygen` 的手册，可以按键盘上的 `Q` 来退出，或按键盘上的 `H` 来进入 `man` 的帮助页面。在帮助页面上可以找到一些常用功能的快捷键。

    很多通过终端来交互的应用都会提供对应的手册，如果不知道某样东西怎么用，可以效仿这种方式来找到它的手册读一读。

    ??? warning "在每次生成密钥对之前"
        请检查用户家目录下的对应位置是否已经有一组对应的密钥：
        ```bash
        ls ~/.ssh
        ```
        如果已经存在文件名为 `id_ed25519` 和 `id_ed25519.pub` 的密钥，在使用 `ssh-keygen` 在同一位置生成密钥时会询问是否覆写。 如果在生成新密钥时覆写掉原来的密钥，可能给利用原来密钥进行认证的服务带来麻烦。

    请你阅读手册，生成一对注释为自己电子邮箱地址的 Ed25519 密钥，并保存到默认位置 `~/.ssh` 目录下。
    
    你可以通过在终端中执行 `ls ~/.ssh` 来验证自己的密钥是不是正确生成，正确生成的标志是这一目录下存在文件名为 `id_ed25519` 和 `id_ed25519.pub` 的文件。

    其中前者叫作私钥，绝对不应该泄露，后者是公钥，可以公开，需要拷贝到需要认证我们身份的服务器上。

    ??? question "我应该怎么设置 passphrase？"
        passphrase 是一个用于加密本地 SSH 私钥文件的密码，在 SSH 连接时，ssh 会先使用 passphrase 解密本地的 SSH 私钥，再使用这一私钥来进行数字签名。当前 `ssh-keygen` 生成密钥时默认的 passphrase 为空。可以考虑设置一个足够长并且自己记得住的 passphrase，这可以提高使用 SSH 的安全性。如果觉得每次连接时输入 passphrase 太麻烦，可以使用 OpenSSH 套件中的 `ssh-agent`，具体使用方法请自行了解。

我们一般把私钥存放在 `~/.ssh` 目录下。`ssh` 在连接时默认会尝试 `~/.ssh` 下的若干固定文件名的私钥。大家可以在使用 `ssh` 时带上 `-v` 这一命令行参数来查看具体尝试了哪些。

!!! info "你知道吗？"

    `~/.ssh` 是 `ssh` 的用户级密钥配置文件目录，其中 `~` 是用户家目录的简写。

    在这个目录当中存放着 `ssh` 运行时会读取的各类配置。

    除了用户级配置目录之外，还有系统级配置文件目录，位置在 `/etc/ssh`。每次运行 `ssh` 时，它都会按照命令行参数、用户配置、系统配置的优先级降序决定本次运行的配置。许多类 Unix 软件都有这样的配置文件目录结构。
  
如果发生私钥泄露、passphrase 遗忘等情况，我们需要使用 `ssh-keygen` 重新生成密钥对并重新分发公钥。在使用新公钥前，应当先在你使用 SSH 连接的所有服务器上彻底删除与泄露私钥对应的公钥。

### 把公钥分发到服务器

公钥认证要求在连接之前把我们的公钥拷贝到要连接的服务器上。

对于类 Unix 系统的远端设备，我们需要将自己的 **公钥** 拷贝到远端设备上对应用户家目录下的 `~/.ssh/authorized_keys` 文件当中。一个比较方便的办法是使用 OpenSSH 套件中的 `ssh-copy-id`，大家可以在有需求时自行了解。`authorized_keys` 是 SSH 服务端允许登录的公钥清单，具体格式记录在 `sshd` 的手册中。

对于其他的云服务商，我们可能需要阅读厂商特定的文档来了解如何将自己的公钥分发到服务器。

!!! note "以 GitHub 为例"

    1. 在 GitHub 的任意页面单击页面右上角的头像。
    2. 单击下拉菜单中的 “Settings”。
    3. 在页面左侧的边栏中单击 “Access” 下的 “SSH and GPG keys”。
    4. 单击页面右侧的 “New SSH key”。
    5. 在 “Title” 中给密钥取一个名字。
    6. 确认 “Key type” 为 “Authentication Key”。
    7. 使用 `cat ~/.ssh/id_ed25519.pub` 来查看刚刚生成密钥对中的 **公钥** 文本并将其复制粘贴到 “Key” 中。
    8. 点击 “Add SSH Key”。
    
    GitHub 也提供了[一个文档](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)来讲解这一过程。

### 连接到服务器

在将公钥分发到服务器后，我们可以在终端中使用 OpenSSH 套件的 `ssh` 进行连接，基本的命令是 `ssh user@hostname`。

其中 `hostname` 指的是服务器的主机名，常见的形式是一个 IP 地址或是一个域名。而 `user` 指的是服务器上你想要连接的用户。

在不使用其他命令行参数的情况下，客户端会首先在服务器的 TCP 22 端口上建立 SSH 连接。首次连接时会先提示核对主机指纹（见下）。之后才进行用户认证。在认证成功后它会请求一个终端并在其中打开 Shell。

!!! note "继续以 GitHub 为例"

    1. 在终端中执行下述命令来连接：
    ```bash 
    ssh -T git@github.com
    ```
    GitHub 的主机名叫作 `github.com`，用户名是 `git`。
    由于 GitHub 使用 SSH 主要是为了 Git 的数据传输，它并不会给我们分配一个服务器上的终端，因此我们使用 `-T` 命令行参数以不请求分配终端。

    2. 在首次与一台新服务器建立 SSH 连接时会显示：
    ```bash
    > The authenticity of host 'github.com (IP ADDRESS)' can't be established.
    > ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
    > Are you sure you want to continue connecting (yes/no)?
    ```
    我们应当手动核对这一指纹和 SSH 服务器的主机公钥指纹是否一致[^5]。如果一致，输入 `yes` 并按下回车。
    成功连接后，应当显示下述信息[^6]：
    ```bash 
    Hi ryannyui! You've successfully authenticated, but GitHub does not provide shell access.
    ```

    3. 在这之后，当我们使用 `git clone git@github.com:user/repo.git` clone 一个仓库并在其中执行 `git pull`，`git push` 等命令时，Git 也会使用 `ssh` 来连接到服务器并传输数据。

    如果感兴趣，可以使用 `git clone git@github.com:git/git.git` 来 clone 下 git 的源码仓库并查阅当中的 `connect.c`。

    [^5]: 对于 GitHub，可以参见此[文档](https://docs.github.com/zh/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)。
    [^6]: 这里 `ryannyui` 是我的用户名，你应当会在此处看到自己的用户名。

!!! info "主机公钥指纹"
    主机公钥和我们刚刚为了进行公钥认证而拷贝到服务器的公钥不是一回事。主机公钥用于 SSH 传输层协议建立时客户端对服务器身份的验证，而我们刚刚拷贝的公钥用于用户认证协议中服务器对客户端的认证。

    在首次连接输入 yes 后，ssh 会将对应主机的公钥存入 `~/.ssh/known_hosts`，如果在此后连接中公钥发生变化，`ssh` 会拒绝连接，这是为了防范中间人攻击的风险。

    当然，可以造成指纹变化的原因还有很多。当 `ssh` 因为指纹不匹配而拒绝连接时，较好的安全实践是先弄清指纹变化的原因再删除 `known_hosts` 中的对应行，比较好的办法是使用 `ssh-keygen -R`，具体使用方法可以在 `man ssh-keygen` 中查询。

除了 `-T`， `man ssh` 中还记录了大量我们可以使用的命令行参数，大家在有需要时应当优先查阅。

比如假设服务器在 33273 端口上监听 SSH 服务的话，我们可以使用 `-p 33273` 来连接服务器的 TCP 33273 端口。

VS Code 的使用者或许想要参考其[文档](https://code.visualstudio.com/docs/remote/ssh)来使用扩展市场中的 [Remote SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) 插件。
