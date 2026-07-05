---
title: Ubuntu回滚Rust `cp`功能，此前该功能破坏了Live镜像构建
date: 2026-07-05 18:32:53
tags: linux, news
---
Ubuntu向Rust Coreutils的过渡又遇到了一个兼容性问题，这次影响到了重要的Unix命令`cp`。

在最新的Ubuntu基金会团队更新中，Canonical工程师Benjamin Drung报告称，团队已在Ubuntu 26.04 LTS上针对Rust Coreutils 0.8版本验证了Zellic对Rust Coreutils审计中提出的修复方案。然而，Ubuntu已将`cp`命令恢复为GNU Coreutils，并向上游提交了[底层修复的PR](https://github.com/uutils/coreutils/pull/13258)。

该问题在Launchpad上被追踪，编号为[bug #2158691](https://bugs.launchpad.net/ubuntu/+source/coreutils-from/+bug/2158691)，涉及Ubuntu的`coreutils-from`软件包，该软件包决定命令使用的是GNU Coreutils还是基于Rust的`uutils`实现。

根据bug报告，Ubuntu在`coreutils-from 0.0.0~ubuntu26`版本中短暂地恢复了Rust Coreutils版本的`cp`命令。这一更改导致`livecd-rootfs`出现故障，`livecd-rootfs`是负责构建Ubuntu Live镜像的软件包之一。该漏洞被标记为“严重”，直接解决方案是恢复GNU `cp`。

技术问题在于uutils `cp`如何处理归档和符号链接相关选项的组合，特别是像`cp -afL`这样的情况。在GNU `cp`中，`-a` 选项等同于`-dR --preserve=all`，后者包含递归复制和属性保留。

然而，Rust 实现对某些互斥标志的处理过于激进。当像`-a`和`-L`这样的选项组合使用时，`-a`隐含的递归行为可能会丢失，导致诸如“未指定`-r`；省略目录”之类的错误，即使归档模式应该包含递归复制。

需要注意的是，这并非Rust Coreutils整体的重大缺陷。这只是一个细微但令人烦恼的兼容性差异，只有在数十年的脚本、构建工具和发行版基础设施遇到对非常古老的命令行行为的重新实现时才会显现出来。

因此，对于最终用户而言，无需恐慌。Ubuntu不会在稳定版本中发布有缺陷的`cp`，而回退到 GNU `cp`正是开发过程中预期的保守做法。

---

转译自：[Linuxiac](https://linuxiac.com/ubuntu-reverts-rust-cp-after-it-breaks-live-image-builds/)
