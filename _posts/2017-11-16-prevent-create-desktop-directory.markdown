---
layout: post
title: 防止火狐生成Desktop文件夹
categories: howto
tags: [firefox, glitch, linux]
image: /assets/prevent-create-desktop-directory/feature.png
description: 
---

## 引子

近日发现Linux家目录中总会“**自动**”生成一个叫`Desktop`的空文件夹，很是烦人。在各类常用软件中排查了很久，最终发现是**火狐浏览器**在启动时生成了它。然而软件内并没有任何关于Desktop文件夹的设置条目。

## 解决方法

因为火狐参考了*XDG Base Directory*规则，所以使用`user-dirs.dirs`文件指定**XDG_DESKTOP_DIR**即可。

{% highlight shell %}
touch ~/.config/user-dirs.dirs  #创建文件
echo XDG_DESKTOP_DIR=\"$(echo $HOME)\" >>~/.config/user-dirs.dirs #指定XDG_DESKTOP_DIR为家目录
{% endhighlight %}

## 为什么要有Desktop

Firefox有一个功能，将网页中的链接拖拽到桌面上，能自动生成一个快捷方式。具体功能可在`about:config`的`browser.shell.shortcutFavicons`配置。[官方说明网页][moz-howto-shortcut]。

## 创建Desktop的逻辑

{% highlight c++ %}
  if (dir) {
    rv = NS_NewNativeLocalFile(nsDependentCString(dir), true,
                               getter_AddRefs(file));
    free(dir);
  } else if (Unix_XDG_Desktop == aSystemDirectory) {
    // for the XDG desktop dir, fall back to HOME/Desktop
    // (for historical compatibility)
    rv = GetUnixHomeDir(getter_AddRefs(file));
    if (NS_FAILED(rv)) {
      return rv;
    }

    rv = file->AppendNative(NS_LITERAL_CSTRING("Desktop"));
  } else {
    // no fallback for the other XDG dirs
    rv = NS_ERROR_FAILURE;
  }
{% endhighlight %}

使用DXR的搜索功能，在`xpcom/io/SpecialSystemDirectory.cpp`文件**第392行**开始，摘录代码片段如上。可见当火狐发现`XDG_DESKTOP_DIR`没有经过配置时，会主动赋值
成`$HOME/Desktop`。对于XDG的其余配置并没有特殊的判断，并在**第419行**进行XDG配置项的目录创建操作。完整代码请参考FYI中提供的链接。

## FYI

- Youtube - [演示Firefox创建链接的快捷方式(Windows, FF58)][ytb-firefox-shortcut]

- Mozilla Support - [How do I stop firefox from recreating unwanted Desktop directory?][moz-support-prevent-desktop-dir]

- ArchLinux Forum - [Firefox: stop making that Desktop directory!][arch-forum-prevent-desktop-dir]

- Mozilla DXR - [源码：SpecialSystemDirectory.cpp][moz-dxr-codesnippet]

- GNOME Wiki - [XDG Base Directory Specification Usage][gnome-wiki-xdg-usage]

- freedesktop.org Specifications - [XDG Base Directory Specification][freedesktop-xdg-spec]

[moz-howto-shortcut]: https://support.mozilla.org/en-US/kb/create-desktop-shortcut-website
[moz-support-prevent-desktop-dir]: https://support.mozilla.org/en-US/questions/985990
[arch-forum-prevent-desktop-dir]: https://bbs.archlinux.org/viewtopic.php?id=66940
[moz-dxr-codesnippet]: https://dxr.mozilla.org/mozilla-release/source/xpcom/io/SpecialSystemDirectory.cpp
[ytb-firefox-shortcut]: https://www.youtube.com/watch?v=4_LjRlqC5gw
[gnome-wiki-xdg-usage]: https://wiki.gnome.org/action/show/Initiatives/GnomeGoals/XDGConfigFolders
[freedesktop-xdg-spec]: https://specifications.freedesktop.org/basedir-spec/latest/