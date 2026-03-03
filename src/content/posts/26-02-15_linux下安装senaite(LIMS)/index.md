---
title: linux下安装senaite(LIMS)
published: 2026-02-15
description: linux下安装senaite
tags: [python, 教程]
category: 技术博客
---

`senaite`是典中典老大难屎山,需要点耐心
建议以本文为主

[官方安装文档](https://www.senaite.com/docs/installation)

我用的是`Mint`,基于乌班图

## 基础软件包

```bash
sudo apt install zsh vim git byobu net-tools tree neofetch
```

```bash
sudo apt install build-essential libbz2-dev zlib1g-dev libssl-dev \
  libsqlite3-dev libffi-dev uuid-dev libnss3-dev libgdbm-dev \
  libgdbm-compat-dev libncursesw5-dev liblzma-dev libreadline-dev \
  libpcre3-dev libcairo2 libpango-1.0-0 libpangocairo-1.0-0
```

**不喜欢zsh和vim可以不装**

## pyenv

```bash
curl https://pyenv.run | bash
# 连接失败的话开代理设置环境变量
# win下cmd里ipconfig获取ip
# 代理软件一般用clash，端口默认7890，看情况
# 将IP和端口替换为你实际的值
export http_proxy="http://192.168.137.1:7890"
export https_proxy="http://192.168.137.1:7890"
# 注意这是临时的,另一个终端就丢了
```

我选择bash

```bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo '[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init - bash)"' >> ~/.bashrc
source ~/.bashrc
# 测试
pyenv --version
```

## 创建环境

```bash
# 这会让你编译,需要前面的基础软件包
pyenv install 2.7.18
```

### 虚拟环境

**记得提前选好位置,路径不要有中文**

```bash
pyenv virtualenv 2.7.18 python2.7-senaite
pyenv activate python2.7-senaite
# 检查是否处于虚拟环境
which python
```

### senaite项目设置

```bash
# 创建目录
mkdir senaite && cd senaite
touch buildout.cfg
```

编辑buildout.cfg
magnitude = 1.0.1一定要加进去,不然这个库最新版并不兼容

```cfg
[buildout]
index = https://pypi.org/simple/
extends = https://dist.plone.org/release/5.2.14/versions.cfg
find-links =
    https://dist.plone.org/release/5.2.14/
    https://dist.plone.org/thirdparty/

parts =
    instance

eggs =
    senaite.lims

eggs-directory = eggs
download-cache = downloads

[instance]
recipe = plone.recipe.zope2instance
http-address = 0.0.0.0:8080
user = admin:admin
wsgi = on
eggs =
    ${buildout:eggs}

[versions]
senaite.lims = 2.5.0
et-xmlfile = 1.1.0
magnitude = 1.0.1
```

创建包含以下内容的 requirements.txt 文件：

```txt
setuptools==44.1.1
zc.buildout==2.13.8
wheel
magnitude = 1.0.1
```

安装依赖

```bash
pip install -r requirements.txt
# 完成后构建,构建时间很长
buildout -c buildout.cfg
```

## 启动

```bash
bin/instance fg
```

如果触发了 `ascii codec can't encode` 错误,是因为路径有中文
把项目挪到合适位置
重新激活虚拟环境

```bash
# 重新运行 buildout（修复路径引用）
buildout -v
# 启动实例
bin/instance fg
```

![](<26-02-15_linux下安装senaite(LIMS)_1.png>)

启动成功,打开`http://localhost:8080`就行
如果失败了看看是不是端口被占用了

然后admin账号登录就行了默认设置就行了
![](<26-02-15_linux下安装senaite(LIMS)_2.png>)
