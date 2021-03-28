---
description: >-
  libmill是Martin
  Sustrik发起的一个go风格协程库，这个协程库其实已经可以满足很多场景的使用了，已经超越了现如今绝大部分c协程库函数的实现了。但是，它还有个更强大的版本libdill，来支持更多的使用场景。不禁想起一句话，”当我们需要一个新特性的时候，我们真的需要发明一个语言吗？”，maybe!
  maybe not!
---

# libdill协程库

libmill作为一个go风格协程库的简单实现，它也是不支持栈空间动态伸缩的，libdill则是在libmill基础上的进一步升级，它支持协程栈大小的动态伸缩，能够适应更多应用场景，在生产环境中使用时，libdill则更值得选择。本书出于学习目的，就以libmill来作为学习材料了，背后的原理也是大同小异的。
