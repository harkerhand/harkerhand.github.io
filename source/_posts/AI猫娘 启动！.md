---
title: 'AI猫娘 启动！'
date: 2025-03-19 15:47:24
tags:
- Python
- NCatBot
categories:
- 经验分享
cover:
index_img: ./AI猫娘 启动！/logo.png
banner_img:
---



# AI猫娘 启动！

NcatBot，基于 NapCat 的 QQ 机器人 Python SDK，快速开发，轻松部署。

# 前言

笔者从朋友了解到，有一个做QQ聊天机器人的项目，只有不到三位数的star，但简单看完其 README 和文档之后，笔者大受震撼！这个项目将QQ聊天Bot的部署成本降低到了**只需要基础Python语法**的地步，得益于其优秀的插件功能，你甚至可以**一行代码不写**，就可以实现诸如课程自动提醒、群聊自动参与聊天等等功能，那么事不宜迟，速速开始！

# 效果展示

一开始就从枯燥的教学开始，太过乏味，买药之前不如先看看疗效

在我们提供的配置下，猫猫的一般最小回复间隔是10秒，如果被@到会强制回答喔

![](./AI猫娘%20启动！/image-20250305221039177.png)

![](./AI猫娘%20启动！/image-20250305221233188.png)

# 基础准备

进入 [NcatBot 文档 | NcatBot 文档](https://docs.ncatbot.xyz/)，点击**快速**开始（我已经等不及了，急急急！）

*这部分内容较为小白向，如果你对自己的技术充分自信，直接跳到第二节即可*

显然，你需要装一个Python，你可以从这里下载 [Python Release Python 3.12.9 | Python.org](https://www.python.org/downloads/release/python-3129/)，（如果你不知道选什么版本的话，直接装这个 [Python312Win64](https://harker.lanzouw.com/iZezN2po6asb) 就可以了。

随后，你需要安装 NCatBot，在命令行执行下面的代码即可，如果你不理解前两行的意思，只执行第三行即可。

```shell
python -m venv .venv
./.venv/Scripts/Activate
pip install ncatbot -U -i https://mirrors.aliyun.com/pypi/simple
```

随后，检查是否安装成功，命令行输入 `python` **之后**输入下面的代码，如果成功输出版本号, 则安装成功，输入 `exit` 回车退出python。

```python
import ncatbot
print(ncatbot.__version__)
```

随后执行

```shell
mkdir ncatbot
cd ncatbot
```

在新建一个 `main.py` 文件，在其中粘贴**文档中**的代码，并把第8行的”123456“改为你要部署机器人的QQ号。

执行下面的代码会运行 `main.py` ，随后会提示你安装napcat，输入N回车即可，随后会弹窗要求你扫码登录（它甚至不需要你输密码，我哭死），登录完成后，Bot应该就正常运行了。

```shell
python main.py
```

向Bot发送“测试”，Bot会回复

![](./AI猫娘%20启动！/image-20250305192930202.png)

那么恭喜你，安装成功了！

# 插件配置

在 `main.py` 同目录下新建文件夹 `plugins`

将 [CatCat](https://harker.lanzouw.com/ioxLW2pp722j) 解压到 `plugins` 目录下，确保目录结构是

```text
ncatbot/
├── plugins/
│   └── CatCat/
│   │   ├── main.py
│   │   ├── requirements.txt
│   |   └── OtherFiles
│   └── OtherPlugin/
├── napcat/
│   └── files
└── main.py
```

执行 

```shell
pip install -r plugins\CatCat\requirements.txt
```

然后在 `plugins\CatCat\config\config.yaml` 中填写你的 `API_key` 和 `plugins\CatCat\config\cat_prompt.txt` 中填写猫娘的 `prompt`，保存后插件就可以正确运行啦！

***注意：你需要把Bot账号的昵称改为”猫猫“或在群内将其昵称改为“猫猫”***

## 深入配置

如果你只是想要一只猫猫聊天，下面就不需要看了喵~

### 关于@

如果你不想更改Bot昵称，在 `plugins\CatCat\main.py` line 50，将其中的“@猫猫”修改为"@\<Nick Name\>"，其中”Nick Name“是Bot的昵称。

### 关于回复延迟

如果你对猫猫的回复延迟感到恼火，认为她经常不回你的消息，在 `plugins\CatCat\main.py` line 40,58 修改对应的数字为0即可，此时猫猫会条条必会（除非她认为你的问题很没价值），另外，注意你的API钱包余额。

### 关于上下文联想

在猫猫睡醒（开机）后，会持续记录群聊信息，在 `plugins\CatCat\main.py` line 66 中的数字规定了最大上下文信息条数；猫猫打盹（关机）后群聊信息会丢失，后续版本会改进。所以目前如果不定期打盹的话，猫猫会累死（内存会炸）。

### 关于富哥

在 `plugins\CatCat\AiChat.py` line26-31规定了调用DeepSeek时的配置，如果你认为自己的钱包非常富裕，那模型的选择和最大token你可以自行修改。后续版本也会持续更新其他语言模型的API适配。

# 结语

至此，你已经成功搭建了一只可爱的 AI 猫娘，并且学会了基本的插件配置与个性化设置。无论是作为一个智能聊天伙伴，还是群聊中的活跃成员，她都能陪伴你度过美好时光！

如果你对 Bot 的功能仍然意犹未尽，不妨尝试编写自己的插件，项目地址 [NcatBot](https://github.com/liyihao1110/NcatBot)。未来版本的更新也可能带来更多的有趣玩法，比如支持更多语言模型、优化上下文管理等，敬请期待喵~

如果在搭建过程中遇到问题，欢迎查阅 [NcatBot 文档](https://docs.ncatbot.xyz/) 或加入相关讨论群 [201487478](http://qm.qq.com/cgi-bin/qm/qr?_wv=1027&k=Vu7KB9gEv9TftvJLYcl846CTqsLUc6Ey&authKey=8JBJhlZro%2B1%2FakeBZ3yJMVeHzlsFYTMHU0RJK%2FpMBmkpZSH7w2CbXU6M2X66PTCQ&noverify=0&group_code=201487478)，与大家一起交流经验。

**那么，祝你和你的 AI 猫娘玩得开心！喵~** 😺🎉