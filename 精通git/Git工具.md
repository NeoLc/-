祖先引用-技巧

`git reset HEAD^`，其中"^"表示第一父提交，注意不是父提交，是“第一”父提交，因为合并分支下，一个提交可能会有多个父提交。

同理，`^^`，表示第一父提交的第一父提交也就是祖父提交。

那么有第一父提交，是不是也有第二父提交呢，答案是有的，"^2"表示第二父提交，"^3"表示第三父提交，注意：不要把"^3"理解为"^^^"的缩写。

除了“^”之外，“~”也可以用于父提交的获取，例如：`HEAD~`表示父提交，`HEAD~~`表示父提交的父提交。

注意：`HEAD~2`表示的可不是第二父提交，它等价于`HEAD^^`，这样看还是推荐使用波浪号。


分支比对-技巧

想知道A分支中，没有合并到B分支的提交有哪些，可以使用“..”双点符，如下：

`git log B..A`   // 显示A分支有，而B分支没有的提交
`git log A..B`   // 显示B分支有，而A分支没有的提交


还有一个非常有用的使用场景，判断本地的代码是否是最新的，如下：

`git log HEAD..origin/master`  // 显示远端分支有，而本地没有的提交，如果显示为空，表示本地代码是最新的。
`git log origin/master..HEAD`  // 显示本地有，而远端分支没有的提交，可以用于了解将要提交哪些修改到服务器。

还可以用于提交冲突判断，非常有用，如下：

`git log HEAD..origin/master`   // 显示非空，说明本地代码不是最新的
`git log origin/master..HEAD`   // 显示非空，说明本地有合并到远端的代码

如何先后执行上面两个命令后，结果都是非空，显然表示在不是最新的代码上进行开发，必然会产生提交冲突。

还有另一种符号，“…”三点符，可以更方便的进行提交冲突判断，如下：

`git log HEAD...origin/master`  // 显示两个分支包含，但是又不是同时包含的分支。
![image](https://github.com/user-attachments/assets/11dde778-98df-45a7-a077-f79e3f4053c6)



如果结果中，两个分支都有显示，此时提交必然会产生冲突。

PS：还有更复杂的分支比对技巧，eg：`git log refA refB --not refC`或者`git log refA refB ^refC`，使用场景较少，了解即可。

交互式暂存

![image](https://github.com/user-attachments/assets/ea064052-8e30-4799-951d-f35ef9b046e9)


交互式操作（-i/interactive），使用场景较少，了解即可。

存储和清理

git stash         // 开始存储，不存储未跟踪的文件，存储暂存区的状态.，分支切换时，会用到该命令。
git stash -u -k   // -u(include-untracked)存储未跟踪文件，-k(keep-index, 这里index就是暂存区)不存储已暂存文件，部分提交时经常用到该命令。
git stash --patch // 交互式暂存, 使用场景较少.
git stash         // 显示存储栈
git stash apply   // 应用一个存储
git stash dpop    // 丢弃一个存储
git stash pop     // 从栈上pop出一个存储
git stash branch  // 从存储时提交为基，创建分支，可以用来解决合并存储冲突问题
 
git clean        // 清理未忽略的未跟踪文件
git clean -f -d  // -f(强制)，强制清理所有未跟踪文件及空目录
git clean -n     // 清理演习
git clean -x     // 忽略的文件也清理掉，例如：清理build下的编译生成文件(.o)确保完全干净的构建.
git stash --all  // 更安全的清理手段

疑问：
+ 存储栈保存在哪里？在`.git`目录下嘛？
	- 会保存在`.git/refs/stash`目录下.
	- ![image](https://github.com/user-attachments/assets/e470a8bf-5897-4102-934c-c871f1df6e5e)

	- refs的stash中存储的是一个hash-id，貌似是新创建的对象，具体的原理未知。

	![image](https://github.com/user-attachments/assets/f02ec523-9289-4457-82d2-18520ad95e53)


	
+ 存储栈是每个分支共享的吗？
	- 经试验，是的。
	
+ 暂存区是每个分支共享的吗？
	- 当然啦，暂存区只有一个(.git/index)

签署工作

有点类似于安全校验，A签署提交，B验证签名后合入，原理未知，猜测是公钥私钥之类的。

使用场景较少，了解即可。

搜索

git grep -n "xxx"          // -n(显示行号)，git的搜索要比linux grep快很多.原理未知，猜测一方面因为只搜索跟踪文件，另一方便是.git存储是结构化的.
git grep -n -e href -e li  // 多关键字搜索

除了直接内容搜索外，git还可以进行关键字与日志的关联搜索，比如：搜索某个变量是什么时候添加到文件的，如下：
![image](https://github.com/user-attachments/assets/f7156c2c-9afa-4e5d-80de-fd02a9036b14)


-S：显示新增和删除该字符串的提交，是非常有用的命令。
还可以使用`-G`选项，实现正则表达式搜索，例如：`git log -GSYS_BIG_BATTERY_*`

Git还具备更强大的搜索功能，可以显示某个函数的修改历史，例如：`git log -L :updateValues:BatteryMonitor.cpp`，表示BatteryMonitor.cpp中updateValues的修改历史。

![image](https://github.com/user-attachments/assets/9fc82cca-3f46-422a-a37e-0885bd39ab99)


重写历史

前面在Git分支章节，作者千叮咛万嘱咐，不要修改历史，在本章节还是介绍了通过`git rebase`修改提交历史的方法。

使用场景较少，了解即可。

Git基本操作~内部解密

本小节非常重要，比较直观的介绍了Git基本操作过程中，看不见的内部到底发生了什么。

git的操作，本质上是操作几个文件，包括：
+ `.git/index`（暂存区-可以理解为下一次提交的缓存）
+ `.git/HEAD`（当前分支引用，指向了最新的提交），很多时候，HEAD就是“git版本库”的“代言人”or“替身”，HEAD的变化，会给人一种版本库发生变化的感觉。
+ `.git/objects`(版本库-感觉叫数据区更合适些)
+ others：工作区(当前目录)、`.git/log`、`.git/refs`等等.

我们以实际的Git操作来直观的感受下这几个文件的变化。

1、当前目录下有一个`file.txt`文件，此时执行`git init`操作，git的内部变化如下：
![image](https://github.com/user-attachments/assets/f9571a2d-d0a8-47e6-b20c-30ed0550b495)



可以看到除了创建一个Git仓库(.git)外，只有工作区有内容，其他文件几乎都是空的.

此时执行`git status`，会提示`file.txt`处于未跟踪(untracked - 红色)的状态.


2、此时执行`git add`命令，添加和跟踪这个文件，git的内部变化如下：
![image](https://github.com/user-attachments/assets/bc5e9607-da93-43be-acdd-9da2e1fdc9dd)



执行git add命令后，暂存区/数据区发生了变化，工作区的内容以一定的格式填充到暂存区中，HEAD未发生变化（没有提交对象，指向谁啊）。

git内部通过`hash-object`命令生成blob对象，通过`update-index`命令写入内容到暂存区。

可以通过`git ls-files -s`查看暂存区的内容.

此时执行`git status`，会提示`file.txt`处于待提交(changes to be committed - 绿色)的状态.

3、接着执行`git commit`，提交该文件，git的内部变化如下：

![image](https://github.com/user-attachments/assets/c41e0fdc-9353-4c1d-8bda-034be14b9908)


此时执行`git status`，会提示(nothing to be commit)状态，工作区、暂存区、数据区（版本库）三个区保持一致。

4、我们修改文件，然后重新走一遍上述流程，第二次提交该版本文件，git的内容变化如下：

![image](https://github.com/user-attachments/assets/a5e5a5d4-759d-4098-9d0c-93d80722b82b)
![image](https://github.com/user-attachments/assets/6f65b27c-aa5a-4e75-8be9-02374f2b85a4)





重置操作~内部解密

本小节介绍了常见的`git reset`的内部实现，假设当前已经有三个版本的file.txt。

+ 执行`git reset --soft HEAD`回退到第二个版本，git内部变化如下：
![image](https://github.com/user-attachments/assets/f226f86f-5afe-4f9f-a88c-b978ca67daf5)



可以看到，HEAD指向的分支还是master，不过master分支的最新提交，从file.txt V3，变为了file.txt V2.
同样是改变HEAD的值，`git reset`会移动当前分支的所指向的提交，`git checkout`分支切换操作，会改变HEAD所指向的分支.

注意：`--soft`选项的作用是回退(回滚)一次提交(git commit)，并不会改变暂存区和工作区。
此时执行`git status`，会提示`file.txt` V3 处于待提交(changes to be committed - 绿色)的状态.


执行`git reset HEAD`回退到第二个版本，git内部变化如下：
![image](https://github.com/user-attachments/assets/f29428fc-9358-4420-9681-24c2ebe131c6)


上图是默认`git reset`命令的内部变化，该命令下，HEAD和暂存区都会因为回退重置，发生变化，也就是回退/回滚`git add`和`git commit`操作。
此时执行`git status`，会提示`file.txt`处于未跟踪(untracked - 红色)的状态.


执行`git reset --hard HEAD`回退到第二个版本，git内部变化如下：
![image](https://github.com/user-attachments/assets/d4b25cef-1938-4c53-ab62-b12121c703e1)

上图是`git reset`命令的内部变化，该命令下，HEAD、暂存区、工作区都会因为回退重置，发生变化，也就是回退/回滚`文件修改`、`git add`、`git commit`操作。
此时执行`git status`，会提示(nothing to be commit)状态，工作区、暂存区、数据区（版本库）三个区保持一致。

总结上面的重置操作：
+ `git reset --soft HEAD^`,重置回退`git commit`操作
+ `git reset HEAD^`, 重置回退`git commit`和`git add`操作
+ `git reset --hard HEAD^`，重置回退`git commit`和`git add`和`本地修改`操作，该命令会覆盖本地修改，小心谨慎使用。


除了可以使用`git reset`操作HEAD外，常见的用法还有重置单个文件，例如：`git reset file.txt`
按照对`reset`的理解，它的作用是回退file.txt的添加(git add)操作，如下：
![image](https://github.com/user-attachments/assets/1dd92b15-533c-468d-9725-fd2ce9890086)


`git reset HEAD <file>`的本质是：将HEAD中对应的<file>，复制到暂存区，同理：
`git reset eb43bf8 <file>`表示的是将`eb43bf8`对应的<file>，复制到暂存区，如下：
![image](https://github.com/user-attachments/assets/8ba57ac4-395f-4d26-b859-3444013cbf56)




感觉复制(替换/覆盖)这个角度，可以更加容易理解`git reset`这个操作，如下：
+ `git reset --soft HEAD^`表示：使用`HEAD^`的内容，替换/覆盖HEAD。                     PS：一般用于提交信息回退修改，压缩提交等场景。
+ `git reset HEAD^`表示：使用`HEAD^`的内容，替换/覆盖HEAD和暂存区
+ `git reset --hard HEAD^`表示：使用`HEAD^`的内容，替换/覆盖HEAD、暂存区、工作区.

重置操作~压缩提交

`git reset`还有一个应用场景：压缩提交。

假设当前有三笔提交，如何将其压缩为一笔呢？如下：
![image](https://github.com/user-attachments/assets/04653819-1db1-4f58-a500-6ab92df36794)



操作很简单，只需要执行`git reset --soft HEAD~3`，然后重新执行`git commit`即可，如下：
![image](https://github.com/user-attachments/assets/80b6b80f-6d26-447c-af49-1337f1a285aa)



重置操作~对比检出

`git checkout`和`git reset`的区别？

场景1：分支切换

`git checkout branch`和`git reset --hard branch_name`的结果非常类似，都改动了三个区的内容。

但是区别是：
+ `git checkout`更改了HEAD本身，`git reset`更改了HEAD所指向的值.
+ `git cecckout`在切换分支前，会检查当前分支的暂存区和工作区是否有未提交的修改，避免覆盖丢失，而`git reset`就没有这个检查了.

场景2：文件检出

`git checkout file`和`git reset --hard HEAD file`结果非常类似

这里有个疑问：`git checkout`的实现原理是啥？和`git reset`非常相似，只是细节步骤不同吗？

![image](https://github.com/user-attachments/assets/60c3e2d4-c458-44cf-a6b7-f0b98dac4dbc)



高级合并

PS: 本章节的本意是介绍一些更有力的工具，来更高效的处理合并问题，但是明显没有起到这个作用。

PS：不太明白为啥一直在说空白冲突处理，感觉是很小众的冲突场景，很多技巧感觉即难用，又没啥太大威力。

`git merge --abort`可以终止合并，回退到合并前的状态。

`git merge`可以加一些参数，影响合并策略，例如：`git merge -Xignore-space-change whitespace`会忽略掉空白符导致的冲突。

`git reset --hard HEAD^`可以撤销合并，缺点是会修改历史。

`git revert -m 1 HEAD`可以还原合并，其中`-m 1`用于指定父节点(合并会导致出现多个父节点的情况发生)，优点是会保存历史，缺点是会影响后续的合并。
revert的原理本质上和reset类似，reset是回退到上一次提交，revert是在合并基础之上再添加上一次提交，这样看起来就像没有合并一样。
打个比方：给车上漆，车原来是红色的，上蓝漆，结果涂失败了，reset操作是将蓝漆都铲掉，变回初始的颜色，revert操作时重新再蓝漆上再涂上红漆，变回初始的颜色。
![image](https://github.com/user-attachments/assets/69482574-8a8e-4430-97e7-02e5467b938c)




关于revert的缺点，书中的解释和日常的revert操作并不一致，日常的revert操作之后，并不会影响后续的分支合并操作，不需要额外的再执行一次`git revert HEAD`操作，
我想可能gerrit的revert和本地的revert还原操作不一致吧，总之我自己不太推荐本地使用revert操作.

![image](https://github.com/user-attachments/assets/aa0c6b2a-cff8-4522-825d-737edd77963e)
![image](https://github.com/user-attachments/assets/0d97f340-14ec-489e-a798-1c11910a3e78)




永远不会产生冲突的合并大招：直接告诉git，当发生冲突时，应该选择合并哪个分支的修改，方法为：`git merge -Xours branch`(合并我们的)，`git merge -Xtheris branch`

本小节还介绍了子树合并，场景较少，了解即可，不过关于添加远程仓的操作（`git remote add remote_name URL`），还是有点陌生，待加强。

Rerere

`rerere`是`reuse(再次使用) recorded(记录) resolution(解决)`的简写，字面意思是：再次使用已记录的解决方案。
使用场景是记录冲突解决，下次遇到相同的冲突时，git自动解决它。

大概看了一下，感觉使用场景较少，了解即可。

使用Git调试

文件标注：`git blame`可以查看文件中的每一行的最后修改时间，以及是被谁修改的。
二分查找：`git bisect`可以通过二分查找，进行bug查找。                            PS：用的比较少，一般是二分法验证版本。

子模块

PS：子模块(`git submodule add URL`)和`git add remote`有啥区别吗？

子模块允许将一个Git仓库作为另一个Git仓库的子目录，感觉还是挺有用的。

PS：但是实际工作中，会直接拷贝第三方源码，有更新再手动打patch，不是第三方的话，也会将多个项目合并到一个大仓中，协作开发。

实际工作中，使用场景很少，了解即可。但是直觉上，这明显是很有用的一个功能，可以避免手动更新。

```
git submodule add url // git会将项目放在一个与仓库同名的目录(空的)
git submodule init   // 初始化
git submodule update // 抓取数据
```

打包
有个场景，当断网或者因为内网限制，无法提交代码时，如何将自己的本地修改分享给其他同事呢？

最常见的思路是生成patch，然后邮件发送或拷贝给同事，比如：`git format-patch  HEAD~3..HEAD`，该命令会生成三个patch补丁，通过执行`git apply`可以应用补丁.

通用格式：`git format-patch [start-point] [end-point]`，point点可以是分支、标签、commit-hash，当然本质上都是哈希值。

生成补丁的常见命令(`git diff/git format`)：
```
git format-patch origin/master..HEAD   // 生成一系列补丁，表示相对origin/matser分支，当前分支所有未推送的提交，即：from origin/master HEAD to current branch HEAD.
git forate-patch -5 master             // 生成5个patch，从matser分支的HEAD到HEAD~5.
```
![image](https://github.com/user-attachments/assets/4d9fa7c4-9e98-4bed-8d4c-831176e44287)




打补丁的常见命令(`git apply/git am`)：
```
git apply xxx.patch            // 打补丁，可能不成功(因为冲突)
git apply xxx.patch xxx.patch  // 打多个补丁
git apply --check xxx.patch    // 检查补丁是否能打成功
git apply --reject xxx.patch   // 遇到冲突并手动解决，不加该选项时，会生成`.rej`文件记录无法自动解决的冲突.
```
`git apply`有两个"缺点"：
+ 处理批量补丁比较麻烦，需要挨个输入补丁名
+ 无法自动生成对应的提交记录，需要手动重新提交.

`git am`可以完美的解决上面的问题，用法为：`git am *.patch`
+ 可以非常容易的批量处理补丁
+ 自生成补丁对应的提交记录

总结下：
+ 未提交的修改，使用`git diff`生成的补丁，对应使用`git apply`打补丁
+ 已提交的修改，使用`git format-patch生成的补丁，对应使用`git am`打补丁.

![image](https://github.com/user-attachments/assets/5eb5deac-8adf-4fbc-8aab-c010843eff5b)



使用`git fomate-patch`会生成一系列的patch，收集和发送都比较麻烦，`git`提供了一个更强大的命令(`bundle`)来解决该问题。
`bundle`字面意思为：捆、一批。是一种将提交打包的技术（终于回到本小节的主题了），它的常见命令如下：

```
git bundle create commit.bundle master ^9a466c5   // 在master分支上, 将9466c5之前的提交打包，文件命名为commit.bundle
git bundle verify commit.bundle                   // 检查打包文件是否合法
git bundle list-heads commit.bundle               // 查看包信息
git fetch commit.bundle master:other-master       // 从包中提取master分支的提交，到仓库的other-master分支.
git merge other-master                            // 合并提交
```
使用`git bundle`进行打包的技术，日常工作中几乎没有人用过哎，比较奇怪，不知道是不是它的使用场景比较少的缘故。

替换

这小节我没怎么看懂，感觉是很高级的技巧，简单的示例用法：`git replace <被替换对象> <替换对象>`，对象可以是提交、标签、树对象的hash值.

一般提交命令用在本地仓库的操作，有一定的危险性，使用场景较少（虽然我不太明白它有啥用），了解即可。


凭证存储

扫了一眼，直觉可以得出，使用场景较少，了解即可。


MISC

PS：有一个小技巧，可以反向的获取引用名对应的哈希值，eg：`git rev-parse HEAD`
![image](https://github.com/user-attachments/assets/6b85950b-261f-42aa-a6e6-06ccb44a5409)


