---                                                                                                        
layout: post
title: "GitHub+Octopress搭建免费blog" 
date: 2012-12-27 11:45
comments: true
categories: 
---

## 生成github公钥 ##

1.  检查ssh公钥设置：

    如果id_rsa*文件不存在，跳到第三步；
    
        $ cd .ssh
        $ ls

2.  备份原来的ssh key：

    备份旧数据，备份后删除旧数据；
    
        $ mkdir key_backup
        $ cp id_rsa* key_backup
        $ rm id_rsa*

3.  生成github ssh key：

        $ ssh-keygen -t rsa -C "id@youremail.com" <github 帐号>
        Generating public/private rsa key pair.
        Enter file in which to save the key (/Users/your_user_directory/.ssh/id_rsa):<回车就好>

    然后系统会要你输入加密串

        Enter passphrase (empty for no passphrase):<github密码>
        Enter same passphrase again:<再次输入密码>

    最后看到下面信息就成功了

        Your identification has been saved in /Users/xxx/.ssh/id_rsa.
        Your public key has been saved in /Users/xxx/.ssh/id_rsa.pub.
        The key fingerprint is:
        ac:63:ff:c9:67:8f:c7:b7:26:43:77:83:bd:ef:11:be username@gmail.com
        The key's randomart image is:
        +--[ RSA 2048]----+
        |                 |
        |                 |
        |                 |
        |       .         |
        |        S       .|
        |       .       ..|
        |      +      .o..|
        |     . o . .o*o==|
        |        ..+ooo*EO|
        +-----------------+ 

4.  添加ssh key到github：

    复制id_rsa.pub的内容到github `Account Settings` -> `SSH Keys` -> `add SSH Key` -> `Key`

## 在github上使用octopress ##

1.  在你本地安装octopress
    
        $ git clone git://github.com/imathis/octopress.git octopress
        $ cd octopress    # If you use RVM, You'll be asked if you trust the .rvmrc file (say yes).
        $ ruby --version  # Should report Ruby 1.9.2

        $ gem install bundler # Install dependencies
        $ bundle install

        $ rake install # Install the default Octopress theme

2.  配置你本地的octopress

    - 创建github repo，名称`username.github.com`

    - 配置octopress使之与`username.github.com`关联

          $ cd octopress
          $ rake setup_github_pages
            Enter the read/write url for your repository
            (For example, 'git@github.com:your_username/your_username.github.com')
            Repository url: <git@github.com:yourname/yourname.github.com>

    以上执行后会要求 read/write url for repository ： </br>
    git@github.com:yourname/yourname.github.com.git

3.  初始化github 

        $ rake generate
        $ rake deploy

    等待几分钟后，github上会收到一封信：“[yourname.github.com] Page build successful”，第一次发布后等比较久，之后每次都会直接更新

4.  将 source 也加入 git

        $ cd octopress
        $ git add .
        $ git commit -m 'your message'
        $ git push origin source

5.  更新 Octopress

    日后有 Octopress 新版本发布，使用以下指令升级。

        $ git pull octopress master     # Get the latest Octopress
        $ bundle install                # Keep gems updated
        $ rake update_source            # update the template's source
        $ rake update_style             # update the template's style

6.  编写并发布文章

    - 发表新文章：

          $ rake new_post["文章名称"]

      该命令会在你的”octopress/source/_posts”目录下生成对应的”.markdown”文件，用任意文本编辑器编辑，使用markdown语法编写你的文章。

    - 生成：

          $ rake generate # generate your blog static pages content according to your input. 

    - 预览：
    
          $ rake preview # start a web server on "http://localhost:4000", you can preview your blog content.

    - 发布：

          $ rake deploy # push your static pages content to your github pages repo ("master" branch)

    - 生成并发布：
      
          $ rake gen_deploy
