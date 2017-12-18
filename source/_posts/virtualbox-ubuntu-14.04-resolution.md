title: virtualbox 安装ubuntu 14.04后分辨率不正确的问题修复
date: 2015-01-22

tags:
 - virtualbox
 - ubuntu
categories:
 - 转载文章
thumbnail:
---

Instead of using the Virtualbox Guest Addition ISO file, try using the following commands in the terminal on the guest Ubuntu Virtual Machine :

Update apt-get. apt-get update

Install dependencies apt-get install build-essential linux-headers-$(uname -r)

Install guest additions apt-get install virtualbox-guest-x11

Use sudo before the commands if required.
