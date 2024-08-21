直觉上直接通过shell git命令也能在程序中使用，但是这种方法有一些缺点：
+ 需要对git的命令行输出文本进行解析，相当麻烦。
+ 需要开新进程，不好管理

通过Libgit2库，以C-like接口的形式，使用Git的能力。

除此之外，还有LibGit2Sharp、objective-Git、pygit2、Jgit(java环境下)
