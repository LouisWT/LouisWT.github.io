---
layout: simple-article
title: 实习开发环境搭建
categories:
    - Mac
tags:
    - Mac
---
6月初就要去实习啦，到时候不想花费太多时间在环境搭建上，因此先提前总结一下。
<!-- more -->

# 1. Mac
安装所有软件
### 1.1 同步 VScode 配置
通过安装 Settings Sync 来同步配置

### 1.2 安装 Homebrew
```s
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

如果没有翻墙
```s
cd "$(brew --repo)"

git remote set-url origin https://mirrors.ustc.edu.cn/brew.git

cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"

git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git

cd "$(brew --repo)/Library/Taps/homebrew/homebrew-cask"

git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git

brew update
```

### 1.3 安装 iterm2
```s
brew cask install iterm2
```

### 1.4 安装 wget
```s
brew install wget
```

### 1.5 安装 oh-my-zsh
```s
sh -c "$(wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"

echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc

source ~/.zshrc

# 想修改主题的话，就在 .zshrc 中修改 ZSH_THEME
```
### 1.6 设置环境变量
```s
# 为了以防万一，这里记一下 mac 中配置环境变量的几个地方
/etc/profile

# node 的环境变量路径就写在这里 /usr/local/bin
/etc/paths

# 这种单用户下的配置，进入了 sudo 就访问不到了
~/.bash_profile

~/.bash_login

~/.profile

# 之后用的是 zsh，配置要写在 zshrc 里才有用
~/.bashrc
```

### 1.7 配置 go 和 node 的环境变量 (默认可能不需要添加)
一般来说，node 的环境变量不用特意设置，它的可执行文件就在 /usr/local/bin 下面
而/usr/local/bin 一般就已经在 /etc/paths 里面了

go 的默认安装路径是 /usr/local/go
所以需要将 /usr/local/go/bin 加入环境变量

# 2. Node 环境配置


### 2.1 安装 n 来控制版本
```s
sudo npm i -g n
```

### 2.2 安装 nrm 来控制仓库地址
```s
sudo npm install -g nrm

nrm add npm http://registry.npmjs.org

nrm add tb https://registry.npm.taobao.org

# 使用 npm 源
nrm use npm

# 使用淘宝源
nrm use tb
```

### 2.3 安装 ts
```s
sudo npm i -g typescript
```

### 2.4 安装 vue-cli
```s
sudo npm install -g @vue/cli
```

### 2.5 安装 Nest.js
```s
sudo npm install -g @nest/cli
```

### 2.6 安装 lerna 解决 package 之间的依赖关系
```s
sudo npm install lerna -g
```