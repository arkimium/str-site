---
title: 适配Arch Linux的Shelly GUI软件包管理器现在允许你从Flathub安装应用
date: 2026-07-05 14:22:54
tags: linux, flathub
---
适用于Arch Linux发行版的pacman替代品Shelly今天更新至2.4.1.1版本，该版本引入了多项新功能和改进，尤其对Flatpak/Flathub的粉丝来说更是如此。

Shelly 2.4.1.1版本在Shelly 2.4.1发布一周后推出，虽然看起来像是一个小小的更新，但实际上它引入了一些令人兴奋的变化，例如可以通过点击“安装”按钮直接从Flathub网站安装Flatpak应用程序。

Shelly 2.4.1.1还改进了对AUR（Arch 用户仓库）软件包的支持，增加了对处理AUR软件包中的多个软件包工件的支持，以及按显式或依赖状态筛选已安装的AUR软件包的功能。

除此之外，新版Shelly还引入了以下功能：支持在可用时优先使用预构建的`-bin`软件包变体而不是从源代码构建；新增了一个用于查看软件包统计信息的窗口；以及支持对通知执行默认点击操作。

此外，此版本重构了Flatpak命名空间结构，以提高模块化和可维护性，重构了PKGBUILD审查对话框，并在“推荐”和“软件包安装”窗口中添加了对并行软件包和推荐获取的支持，以加快加载速度。

最后，Shelly 2.4.1.1增加了在搜索结果中选择和突出显示软件包名称和描述的功能，为搜索命令添加了一个`--standard( -ASs)`选项以查询标准软件包，并改进了托盘服务和通知逻辑以减少内存使用量。

此外，它还为与更新相关的CLI命令以及AppImage搜索命令添加了JSON输出支持，使Shelly更易于编写脚本并与其他工具集成，并在安装页面中集成了Starfish。同时，还修复了一些错误，以增强对AppImage的支持。

请查看项目GitHub页面上的发行说明，了解Shelly 2.4.1.1的更多变更详情。您可以从同一位置下载独立的二进制文件或源代码压缩包。您也可以通过运行命令在**CachyOS**上安装Shelly：`sudo pacman -S shelly`。

---

转译自：[9to5Linux](https://9to5linux.com/shelly-gui-package-manager-for-arch-linux-now-lets-you-install-apps-from-flathub)
