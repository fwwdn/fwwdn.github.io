title: Git安装配置
date: 2015-06-07 22:34:23
tags: Tools
---

##安装
windows下git客户端我们选择msysgit;下载地址：https://github.com/msysgit/msysgit/releases/download/Git-1.9.5-preview20141217/Git-1.9.5-preview20141217.exe

##配置
###用户信息

你个人的用户名称和电子邮件地址，用户名可随意修改，git 用于记录是谁提交了更新，以及更新人的联系方式。

`git config --global user.name "test"`

`git config --global user.email "test@sina.com"`

###自动高亮

`git config --global color.ui auto`

###查看所有配置

`git config --list`

##生成SSH Key

1. 查看是否已经有了ssh密钥：cd ~/.ssh
如果没有密钥则不会有此文件夹，有则备份删除；

2. 生存密钥：
`ssh-keygen -t rsa -C "test@sina.com"`
按3个回车，密码为空;最后得到了两个文件：id_rsa和id_rsa.pub

3. 在github上添加ssh密钥,这要添加的是id_rsa.pub里面的公钥。
4. 测试配置：输入ssh git@github.com，如果提示：Hi defnngj You've successfully authenticated, but GitHub does not provide shell access. 说明你连接成功了。

##使用github

1. 获取源码：
`git clone git@github.com:billyanyteen/github-services.git`

2. 这样你的机器上就有一个repo了。

3. git于svn所不同的是git是分布式的，没有服务器概念。所有的人的机器上都有一个repo，每次提交都是给自己机器的repo
仓库初始化：
`git init`
生成快照并存入项目索引：
`git add`
文件,还有git rm,git mv等等…
项目索引提交：
`git commit`
4. 协作编程：
将本地repo于远程的origin的repo合并，
推送本地更新到远程：
`git push origin master`
更新远程更新到本地：
`git pull origin master`
5. 补充：
添加远端repo：
`git remote add upstream git://github.com/pjhyett/github-services.git`
重命名远端repo：
`git://github.com/pjhyett/github-services.git为upstream`

