title: 动态配置多个Git账号
date: 2015-07-01 22:30:45
tags: Tools
---
##前言

因为公司用gitbucket搭建了自己的代码库，要求使用公司的邮箱作为账号。而我自己又有一些开源代码放在github上，所以必须配置多个git账号，下面总结下我的配置过程：

###SSH配置

使用ssh-keygen分别生成gitbucket和 github的密钥：

	cd ~/.ssh/
	ssh-keygen -C "gitbucket email" -t rsa 
	Enter file in which to save the key (/Users/.ssh/id_rsa): id_rsa_gitbucket
	
	ssh-keygen -C "github email" -t rsa 
	Enter file in which to save the key (/Users/.ssh/id_rsa): id_rsa_github
	
上传公钥到gitbucket和 Github服务器。

在~/.ssh目录下创建config文件，进行相应配置：

	#gitbucket
	Host gitbucket
	HostName your_gitbucket_host
	User your_email
	IdentityFile ~/.ssh/id_rsa_gitbucket

	#github
	Host github
	HostName github.com
	User your_email
	IdentityFile ~/.ssh/id_rsa_github

	
###检出服务端项目代码

这里需要注意，使用.ssh目录下的host代替真实的hostname，这样才能让git识别出来,如：

	git clone git@github:fwwdn/sensitive-words.git

这里使用github代替@github.com

###解决commit后不能正常显示头像信息问题

按上面的方法配置后，在你执行git commit -m "XXX"命令时会有如下提示：

	Your name and email address were configured automatically based
	on your username and hostname. Please check that they are accurate.
	ou can suppress this message by setting them explicitly:

    git config --global user.name "Your Name"
    git config --global user.email you@example.com

	After doing this, you may fix the identity used for this commit with:

    git commit --amend --reset-author
    
当然，这并不会影响你push代码，但因为你的user name和你的user email配置有误，所以你的头像将不能在提交日志中正常显示。解决的办法如下：

####1、最笨的办法：

取消git全局设置，很多同学照着网上的教程，都会对git进行全局设置，例如：

	git config --global user.name "username"
	git config --global user.email  "email"

因为我们要配置多个账号，所以我们首先需要取消git的全局设置

	git config --global --unset user.name
	git config --global --unset user.email

针对每个项目，单独设置用户名和邮箱

设置方法如下：

	cd gitbucket/project
	git config user.name "gitbucket_username"
	git config user.email "gitbucket_email"
	
	cd github/project
	git config user.name "github_username"
	git config user.email "github_email"

####2、半自动化方法：

	cd ~
	mkdir .git-scripts
	vim ~/.git-scripts/account-config.sh

shell脚本内容如下：

	#!/bin/bash

	remote=`git remote -v | awk '/\(push\)$/ {print $2}'`

	email="gitbucket_email"
	name="gitbucket_name"

	if [[ $remote == *github* ]]; then
  	  email="github_email"
  	  name="github_name"
	fi

	echo "Configuring user.email as $email"
	echo "Configuring user.name as $name"

	git config user.email $email
	git config user.name $name
	
 然后只需在每次git clone 项目之后，执行**git account-config** ，即可一步配置git account信息 。
 
至此，我们就可以使用多个git账号愉快的玩耍了。:)