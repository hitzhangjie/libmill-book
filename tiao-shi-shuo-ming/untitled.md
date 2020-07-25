---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# 如何调试libmill



osx下面通过`brew install libmill`安装的是共享库libmill.so，很多调试信息都已经被优化掉了，调试起来不是很方便，为了更好地进行调试，可以自己从源码构建安装libmill。

```text
git clone https://github.com/sustrik/libmill
cd libmill

./autogen.sh
./configure --disable-shared --enable-debug --enable-valgrind

make -j8
sudo make install
```

