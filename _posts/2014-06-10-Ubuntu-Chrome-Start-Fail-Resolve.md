---
layout: post
categories: 
  - Linux 
title:  "Ubuntu Chrome启动失败解决方法"
date: 2014-06-10 22:21:35 +0800
comments: true
---

* content
{:toc}

## <a id="Intro">前言</a>

使用`Ubuntu`系统时，由于系统是默认安装的，所以机器名没有修改成一个合适的名字。在`Ubuntu`下通过命令`sudo vim /etc/hostname`来修改成自己想要的命令，然后注销用户，重新进入系统。

接着，就出现了一个很奇怪的问题，我的桌面上的chrome图标不能点击进入，一直都是无响应状态，但是其他浏览器，比如自带的`Firefox`就可以正常启动和使用。

## <a id="Chrome">Chrome启动失败解决方法</a>

点击`chrome`图标不管用，并且看不到任何出错信息，无法定位问题出在哪里。只好，从`终端`使用`google-chrome`命令来启动`chrome`浏览器客户端了。于是，出现以下错误信息：

``` bash
ketao@ketao-Latitude:~$ google-chrome

[5901:5901:0610/183033:ERROR:process_singleton_linux.cc(309)] 其他计算机 (ketao-Latitude-E5430-non-vPro) 的另一个 Google Chrome 进程 (7578) 好像正在使用此个人资料。Chrome 已锁定此个人资料以防止其受损。如果您确定其他进程目前未使用此个人资料，请为其解锁并重新启动 Chrome。

[5901:5901:0610/183033:ERROR:simple_message_box_views.cc(208)] Unable to show a dialog outside the UI thread message loop: Google Chrome - 其他计算机 (ketao-Latitude-E5430-non-vPro) 的另一个 Google Chrome 进程 (7578) 好像正在使用此个人资料。Chrome 已锁定此个人资料以防止其受损。如果您确定其他进程目前未使用此个人资料，请为其解锁并重新启动 Chrome。

```

<!-- more -->

>> Note：显然，上面的错误信息告诉我们，尼玛，`chrome`会记录我们在计算机上的操作数据在本地，然后我们修改主机名后，导致老数据被锁住了，这样新的主机名下chrome无法获取用户对应的浏览器数据，因此chrome无法正常启动，并且在UI上还无法看到相关错误，导致假死现象出现。

### 解决方法：

先把本机上的`chrome`卸载掉，当然这不会把那些被锁定的数据也一并删除掉，不然我也不会写一篇博客记录下。因此，接下来，你需要删除掉自个目录下的残留的`chrome`用户数据。

在当前用户目录下面执行命令：

```bash
ketao@ketao-Latitude:~$ cd .config/
ketao@ketao-Latitude:~/.config$ ls
chromium        gedit             monitors.xml     Trolltech.conf
compiz-1        gnome-session     nautilus         ubuntu-ui-toolkit
dconf           google-chrome     pulse            update-notifier
enchant         gtk-2.0           ReText project   upstart
evolution       gtk-3.0           software-center  user-dirs.dirs
fcitx           ibus              SogouPY          user-dirs.locale
fcitx-qimpanel  libaccounts-glib  SogouPY.users    VirtualBox
ketao@ketao-Latitude:~/.config$ rm -rf chromium/
ketao@ketao-Latitude:~/.config$ rm -rf google-chrome/
```

然后，重装`chrome`浏览器，安装成功之后，就可以启动`chrome`了。

## <a id="Finally">小结</a>

一般，如果程序出现问题，通过重装都无法解决的，都是应用在用户目录下面添加来一些配置文件，并且这些文件在应用卸载之后并不会删除，所以会出现重装失败的问题。
