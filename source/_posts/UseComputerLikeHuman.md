---
title: 用电脑像个人吧
date: 2026-04-23 13:01:34
tags:
categories: TechMagic
index_img: ./UseComputerLikeHuman/logo.png
banner_img:
---


> 人类祖先偶然发现了火，学会了使用工具；后来，他们开始主动的创造工具来满足自己的需求；再后来，他们对于工具的要求不仅仅是能用，还要好用...


那么问题来了，为什么**大部分人类使用计算机的方式还停留在“偶然”阶段**呢？

我们在使用计算机时，往往是被动的接受它提供的功能和界面，而不是主动地去创造和定制它来满足我们的需求。我们习惯于使用预设的软件和操作系统，而不是自己编写代码或定制工具来解决问题。

<p class="note note-warning">
叠甲环节：本文仅从个人的角度出发，分享一些关于如何像人类一样使用计算机的思考和建议。如果对您产生了冒犯或者不适，请按下 ALT + F4。作者也会犯错，如果你有不同意见，可以在评论区攻击我。
</p>


好了，P话结束，正文开始。

# 前言：像人类一样使用AI

询问技术类问题，不要使用豆包、元宝、以及各种手机自带的语音助手。

我建议使用 [ChatGPT](https://chatgpt.com/)、[Gemini](https://gemini.google.com/app)、[DeepSeek](https://chat.deepseek.com/)、[Mimo](https://aistudio.xiaomimimo.com/#/c)。

在使用这些工具时，组织好你的问题，尽量提供足够的上下文和细节。

例如，如果你在编程方面遇到了问题，不要只是说“我的代码有问题”，而是应该描述你正在尝试实现什么功能，遇到了什么错误，以及你已经尝试了哪些解决方法。

下面是一个示例：
```
我在 /home/harkerhand/codes/cpp 路径下，其中有 23  23.cpp 文件，我执行了 g++ ./23.cpp -o 23 命令，但是出现了以下错误：

./23.cpp:5:34: 错误：‘format’ has not been declared in ‘std’
    5 | using std::cout, std::endl, std::format, std::make_tuple, std::string;
      |                                  ^~~~~~
./23.cpp: In function ‘int main()’:
./23.cpp:8:13: 错误：‘format’在此作用域中尚未声明
    8 |     cout << format("The answer to the Ultimate Question of Life, The "
      |             ^~~~~~

其中 23.cpp 的内容如下：

#include <format>
#include <iostream>
#include <tuple>

using std::cout, std::endl, std::format, std::make_tuple, std::string;

int main() {
    cout << format("The answer to the Ultimate Question of Life, The "
                   "Universe, and Everything is {}.",
                   42)
         << endl;
    auto a = make_tuple(static_cast<bool>(1), string("2"), double(3), 4);
    auto [x, y, z, w] = a;
    cout << format("x = {}, y = {}, z = {}, w = {}", x, y, z, w) << endl;
}

我希望我能够成功编译并运行这个程序，输出正确的结果。
```

本文后续内容将会比较简洁，如果你出现任何问题，请按照上面的指示去求助AI工具。

# 像人类一样使用计算机

## 像人类一样使用终端

### Windows

- 别用 CMD 了，用 PowerShell，不仅是 PowerShell，还得是 PowerShell 7。

    [在 Windows 上安装 PowerShell 7](https://learn.microsoft.com/zh-cn/powershell/scripting/install/install-powershell-on-windows?view=powershell-7.6)

- 装好之后，去 Windows Terminal 里把它设置成默认的 Shell。
- 如果你有美化需求，可以安装 [oh-my-posh](https://ohmyposh.dev/)。

### Linux

如果不想装双系统，还需要在 Windows 上使用 Linux 的话，无特殊需求，使用 WSL2，如果遇到其他环境问题，AI 工具应该会告诉你去装虚拟机。

- 放下 Bash，使用 Zsh。
- 装 oh-my-zsh 来美化你的终端并提高效率。
- 装自动补全插件，例如 zsh-autosuggestions 和 zsh-syntax-highlighting。
- 要好看一点装 Powerlevel10k 主题。
  
### 字体

推荐 JetBrains Mono 或 [Maple](https://github.com/subframe7536/Maple-font)，都是开源的等宽字体，支持编程和终端使用。

## 像人类一样使用 SSH

### 配置 SSH Config

配置文件应该在 `~/.ssh/config`，配置后，你可以通过 `ssh servername` 来连接服务器，而不需要每次都输入完整的命令。

### 使用密钥认证而不是密码认证

- 生成 SSH 密钥对，尽量使用 Ed25519 算法。
- 使用 `ssh-copy-id` 将公钥复制到服务器上（或者手动将公钥添加到服务器的 `~/.ssh/authorized_keys` 文件中）。

### 使用 SSH 端口转发

如果有需求，你可以通过 SSH 端口转发来访问服务器上的服务（例如数据库、web服务）或让服务器访问你的本地服务（一般用于让服务器通过本机代理访问外部网络）。

### 使用 SSH Agent 代理密钥

假设你使用ssh密钥来认证github，在服务器中开发时，你需要在服务器中使用ssh连接github来拉取代码或者提交代码，如果你不想在服务器中存储你的私钥，你可以使用SSH Agent来代理你的密钥。

## 像人类一样使用 Git

### 仓库管理

- 使用 GitHub、GitLab 或 Bitbucket 等平台来托管你的代码仓库。
- 代码仓库中应当存储你的源代码、文档、配置文件等与项目相关的内容，而不应该存储编译后的二进制文件、日志文件、临时文件等。
- 使用 `.gitignore` 文件来忽略不需要版本控制的文件和目录，例如编译生成的二进制文件、日志文件、临时文件等。
- 特别的，不要把你的 SSH 私钥或者其他敏感信息存储在代码仓库中。

### 分支管理

- 使用分支来管理不同的功能开发、bug 修复和版本发布。
- 不要一直在主分支（main 或 master）上开发新功能，应该在新的分支上进行开发，完成后再合并到主分支。
- push不上去不要 --force，先 pull 下来看看是什么问题，解决了再 push。
- 如果你需要撤销某次提交，不要使用 `git reset`，而是使用 `git revert` 来创建一个新的提交来撤销之前的更改，这样可以保持提交历史的完整性。
- 如果你需要在本地修改未上传的提交，可以使用 `git commit --amend` 来修改最后一次提交，或者使用 `git rebase -i` 来修改多个提交。

### CI/CD

- 使用持续集成（CI）工具来自动化构建、测试和部署你的代码，例如 GitHub Actions、GitLab CI/CD、Jenkins 等。
- 配置 CI/CD 流水线来自动化执行代码质量检查、单元测试、集成测试、部署等任务，以提高开发效率和代码质量。
- 确保你的 CI/CD 配置文件（例如 `.github/workflows/` 目录下的 YAML 文件）也存储在代码仓库中，以便于版本控制和团队协作。
- 在 CI/CD 流水线中，使用环境变量来存储敏感信息（例如 API 密钥、数据库密码等），而不是将它们直接写在代码或配置文件中。

### 版本管理

- 使用 Git tag 来标记重要的版本，例如发布版本、里程碑版本等。
- 在发布新版本时，创建一个新的 Git 标签，并在发布说明中描述该版本的主要更改和新功能。
- 使用语义化版本控制来管理版本号。

## 像人类一样使用 Docker

### 构建

- 使用 Dockerfile 来定义你的应用程序的环境和依赖。
- 使用多阶段构建来优化你的 Docker 镜像的大小和性能。
- 使用官方的基础镜像来构建你的应用程序，例如 `python:3.10-slim`、`node:18-alpine` 等。
- 在 Dockerfile 中，尽量避免使用 `latest` 标签来指定基础镜像的版本，应该明确指定版本号。

### 运行

- 使用 Docker Compose 来定义和运行多容器的应用程序。
- 使用环境变量来配置你的应用程序，而不是将配置写在代码中。
- 使用 Docker 的卷（volumes）来持久化数据，而不是将数据存储在容器内部。
- 使用 Docker 的网络（networks）来管理容器之间的通信，而不是使用主机网络。
- 定期清理不再使用的 Docker 镜像、容器和卷，以节省磁盘空间。
- 多人同机使用 Docker 时，使用不同的命名空间或者标签来区分不同用户的容器和镜像，以避免冲突和混乱。

## 像人类一样使用键盘

现在是AI时代，你需要习惯性的点击 Tab 键。

你的键盘损耗程度，应该是 Tab > Enter > Space > 其他键。

你应该尽可能在更多的地方使用 Tab 键来触发自动补全功能，这个品味是需要培养的，逐渐的你会产生“这个地方应该可以按 Tab 键”的感觉。