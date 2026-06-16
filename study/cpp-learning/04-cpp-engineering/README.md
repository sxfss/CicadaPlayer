# 04 C++ Engineering

这一组只在 C++ 概念确实帮助理解播放器工程时展开。

推荐顺序：

1. [01-cpp-concepts-through-cicada.md](01-cpp-concepts-through-cicada.md)
2. [02-engineering-patterns-and-lessons.md](02-engineering-patterns-and-lessons.md)
3. [03-cpp-design-review-checklist.md](03-cpp-design-review-checklist.md)

不要把 CicadaPlayer 当成现代 C++ 风格范本直接照抄。更适合学习的是对象职责、接口边界、所有权审查、
生命周期和模块替换点。

`03-cpp-design-review-checklist.md` 是以后读源码和写 AVPlayer 时的审查表：谁拥有对象、谁停线程、错误怎么传、
接口是否过大、哪些历史写法不适合照搬。
