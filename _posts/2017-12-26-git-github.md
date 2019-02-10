---
layout: post
title:  "Git连接GitHub"
date:   2017-12-26
author: Dickie Yang
tags: 
    - git 
---

### 1.生成ssh key
- 可以通过在 Git Bash 中输入ssh命令，查看本机是否安装 SSH。输入ssh-keygen -t rsa命令，
表示我们指定 RSA 算法生成密钥，然后敲三次回车键，期间不需要输入密码，之后就就会生成两个文件，
分别为id_rsa和id_rsa.pub，即密钥id_rsa和公钥id_rsa.pub. 对于这两个文件，其都为隐藏文件，
默认生成在以下目录：<br>
`C:\Users\Administrator\.ssh`<br>
密钥和公钥生成之后，我们要做的事情就是把公钥id_rsa.pub的内容添加到 GitHub，这样我们本地的密钥id_rsa
和 GitHub 上的公钥id_rsa.pub才可以进行匹配，授权成功后，就可以向 GitHub 提交代码啦！
### 2.添加ssh key
- 登录GitHub，进入setting页面后，再点击SSH and GPG Keys进入此子界面，然后点击New SSH key按钮，
我们只需要将公钥id_rsa.pub的内容粘贴到Key处的位置（Titles的内容不填写也没事），然后点击Add SSH key 即可。
### 3.验证是否绑定成功
- 在我们添加完SSH key之后，也没有明确的通知告诉我们绑定成功啊！不过我们可以通过在 
Git Bash 中输入ssh -T git@github.com 进行测试。