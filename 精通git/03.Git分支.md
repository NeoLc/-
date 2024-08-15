介绍分支之前需要了解git是如何保存的，它是理解分支的前置知识。

在前面章节又介绍过快照的概念，它是Git这个分布式版本控制系统和集中式版本控制系统在数据保存方面的基本区别。

在.git目录下，objects保存了版本的核心数据，这些数据分为三类：
	• 提交对象(commit object - 保存数对象索引和索引提交信息)
	• 树对象(tree object - 保存目录结构和blob对象索引)
	• blob对象(保存文件快照)

这是数据以校验和(哈希值)为名进行保存，以哈希值的前两位为目录名，哈希值的剩余位数为文件名，如下图所示：

![image](https://github.com/user-attachments/assets/5d8fc14a-9839-42ab-91c3-792ecd21f9e1)
![image](https://github.com/user-attachments/assets/ed8d8e1e-aa34-4aeb-bbee-66befecd15b9)


接着继续介绍下这三类对象是何时产生的，git的基本工作流程是：修改文件 -> 添加修改到暂存区 -> 提交修改到版本库。

在执行`git add`操作时，git会计算生成修改文件的校验和，并且将修改文件的快照，按照前面介绍的形式，保存在objects目录下。

在执行`git commit`操作时，git会计算生成目录树（保存目录结构等信息）对应的校验和，按照前面介绍的形式，保存在object目录下。

同时会生成提交对象数据（保存提交msg，作者的姓名/邮箱等信息），同样生成校验和，按照前面介绍的形式，保存在objects目录下。


三类数据的生成和保存的简单过程就是这样，不过在object目录下，三类数据混杂，git如何区分那几个目录是和同一笔提交相关联呢？

这里要说一说，这三类数据的数据结构及他们之间的关联。

在提交对象（commit object）中，有一个tree对象的指针指向本笔提交对应的目录树结构，同时还有一个父指针，指向了前一笔提交对象，这样不同提交对象之间通过指针进行关联。

提交对象和tree对象之间同样通过指针进行了关联，而tree对象内部也有一个指针，指向了文件快照，即blob对象，这样tree对象和blob对象也关联了起来。

也就是说，通过一笔提交相关的所有内容，都通过三个结构体以及结构体内部的成员指针串起来，形成一笔完整的提交记录数据。

![image](https://github.com/user-attachments/assets/a6ce1295-35e5-466f-8be6-3b8cd6dfbd45)

![image](https://github.com/user-attachments/assets/5d533244-9370-4350-8265-f33c91b518e0)


了解了Git中的数据保存，尤其是提交对象及指针的作用，对于后面分支操作的理解，大有裨益。

Git的分支操作

传统版本控制管理系统的分支操作非常昂贵，因为它是通过整体拷贝创建副本的方式实现的，耗时耗力，而同样的分支操作，Git却极为快速便捷。

不知道大家有没有在意过，在执行git log操作时，会在最新提交的commit-id后面看到一个HEAD关键字，该关键字本质是一个指向提交对象的指针。

![image](https://github.com/user-attachments/assets/41e4f750-2189-4b03-a41f-d4c77273f5c1)

Git的分支同样也是如何，它的本质也是一个指向了某个提交对象的指针，Git分支的一系列操作，本质就是指针的操作，因此效率极高。

	• 分支的创建

通过git branch branch_name commit-id/tag/branch或者git checkout -b branch_name commit-id/tag/branch/remote_branch创建一个分支

PS: 使用`git checkout -b xxx remote_branch`，会以远端分支(最新提交)为基础，创建跟踪分支, 在分支下, 可以直接使用git pull命令，而不需要添加origin branch后缀, 提交和拉去代码更方便点.

![image](https://github.com/user-attachments/assets/f0cbca5b-478b-463c-b308-9750123c0478)

该操作本质上其实是创建一个指针，指向了对应的提交对象，这个指针的名字就是分支名字。

在新分支提交修改后，该指针会后移，指向最新的提交对象，如下：

![image](https://github.com/user-attachments/assets/fc8b82f3-e222-42c0-ae1a-89a6941f31bc)

大家可以类比链表操作，创建了一个指针，指向了链表的某个节点，整个分支操作和链表指针操作极为类似。

当GIT中有多个分支时，如何判断自己当前在哪个分支呢？也很简单，就是前面介绍过的特殊指针：HEAD。
HEAD指向哪个分支，则表示当前工作在哪个分支上，如果HEAD没有指向任何分支，则处于分离状态，此时HEAD的值为

![image](https://github.com/user-attachments/assets/c51b8586-b9d7-48c4-ad7e-ea1320b5cc2a)

分支切换

通过git checkout命令，可以实现分支的切换，它的本质是让HEAD指向不同的分支指针。

![image](https://github.com/user-attachments/assets/f22c1e32-ef5a-4248-a51c-6b604e6609b4)

分支切换之后，要注意工作区的内容会同步的切换到新分支（即会改变工作区），如果当前分支还有未提交的改动，git会禁止分支进行切换。

牢记：当你 切换分支的时候，Git 会重置你的工作目录，使其看起来像回到了你在那个分支上最后一次提交的样子

	• 分支的删除

通过git branch -d branch_name（删除分支）或者git branch -D branch_name(强制删除未合入的分支)。

在传统版本版本控制系统中，分支的删除会将拷贝的副本删除掉，不仅动作大，而且该分支中的数据全部会丢失掉。

而在git版本控制系统中，分支的删除不过是删除了一个指针，对分支的数据内容毫无影响。


	• 分支的合并

通过git merge操作，可以实现分支合入、单个合入、合并合入等操作。

当开发分支和主分支未发生分叉时，此时分支合并操作非常简单，只需要将master指针平移到开发分支指向的提交对象接口。PS：在主分支合并前，记得更新分支，保证和远端仓库最新提交保持一致。

这种指针右移操作也称为：“快进（fast-forward）”

![image](https://github.com/user-attachments/assets/c56d2af4-9e8e-413b-b32a-3e69cfd2b972)

当开发分支和主分支产生分叉时，在无冲突的情况下，GIT会自动将两个分支在公共祖先之后的提交合并在一起，形成一个合并提交，自动提交到版本库中，合并成功后，通过git log可以查看到。

也就是说git merge做了两个操作：合并（会更新工作区、暂存区） + 提交（会更新版本库），这两个操作都是git自动完成的。

![image](https://github.com/user-attachments/assets/dfadc714-0129-46ff-8d0a-6aa6fdd1335a)

![image](https://github.com/user-attachments/assets/892766ee-fe81-4d11-9620-07ca8d02b145)

PS：运行 git log --oneline --decorate --graph --all，可以直观的查看分支分叉情况，命令太长的话，可以通过前面章节介绍的别名，快速的查看。

冲突是指两个提交对同一个内容，产生了不同的修改，分支合并产生冲突时，需要开发者手动解决冲突后，再手动提交。

![image](https://github.com/user-attachments/assets/b91b13f2-15ee-43b5-96dd-42801c506340)

PS：不要对冲突有畏惧心态，不过是二选一，或者二者融合而已，对代码了解的话，简单的很。


	• 分支查看

运行 git log --oneline --decorate --graph --all，可以直观的查看分支分叉情况

git branch -v  可以查看每个分支的最后一次提交

git branch  可以查看当前分支列表
git branch -r 查看查看远端分支

git branch --meged     查看已经合并到当前分支的分支列表
git branch --no-meged  查看未合并到当前分支的分支列表


远程分支

`git ls-remote`可以查看远程引用。一般可以用该命令，来查看远端的最新的提交，判断自己的本地代码是否是最新的。

![image](https://github.com/user-attachments/assets/a53beb20-2004-4c94-86c8-c30681a10e5c)

通过`git remote show remote_name`可以查看远程分支的详细信息.

远程分支为区分本地分支，命令格式为：(remote)/(branch)，如下：

![image](https://github.com/user-attachments/assets/908c152f-12bc-436c-b3d6-6c574756094d)

上图中origin是远端默认仓库名，在执行git clone时命名，如果想自定义的话，可以使用`git clone -o my_remote_name`

![image](https://github.com/user-attachments/assets/c5024d67-53b1-45ae-baa9-d8181100d18e)

推送代码可以使用`git push origin branch`命令，这个命令其实是被Git简化的，在真正运行时，Git会将branch展开。

例如：git push origin master，git会将master展开为：refs/heads/master:refs/heads/matser。

含义为：使用本地仓库的master分支，来更新远端仓库的matser分支.

其中：refs/heads/master是分支引用，远端的分支引用可以用git ls-remote获取，本地的分支引用查看.git/HEAD可以获取.

变基

Git中，整合不同分支的修改，除了merge操作外，还可以通过rebase。

meger操作和rebase的操作结果是一样的（工作区/暂存区/版本库内容一致），不一样的是提交记录，rebase的提交记录相比meger会更加简洁清晰。

以分叉分支合并为例，如下：

git checkout matser
git merge experiment  // 合并experience分支的修改.

![image](https://github.com/user-attachments/assets/9a514e24-b919-4cf8-8ed8-f4947e1499d4)

git checkout experiment
git rebase matser       // 以master为基，合并当前分支修改.

![image](https://github.com/user-attachments/assets/68ab7feb-7188-4a47-8406-e6508350d92f)

可以看到，checkout到哪个分支，最终哪个分支的HEAD/分支指针发生变化，哪个分支的内容发生了变化。
在内容上，上面两图中的C5和C4'是一模一样的。


rebase操作的原理比较简单，先找到两个分支的共同的祖先(C2)，然后将C4分支相较C2的修改保存为一个临时文件(类似patch)，
然后将分支指针(experiment)指向C3，最后以C3为基，将修改应用过去（类似apply打patch），并自动提交形成C4'(我猜的)，experiment指针自动移动到C4'.

感觉我们日常的git diff生成patch，和apply打patch，就是手动版的rebase，原理大差不差。


一个更加复杂场景的的变基操作，如下图：

![image](https://github.com/user-attachments/assets/4760f270-a164-46c0-87f9-003bc413d907)

只需要一行命令即可实现：`git rebase --onto master server client`

该命令的含义是：取出client分支，寻找client在与server共同祖先之后的修改，并将这些修改，在master分支上“重演一遍”。

这个操作可以轻松的实现截取一个提交链中的部分提交节点，将它们合入到其他的分支。

回到上图的操作，执行后的结果如下图：

![image](https://github.com/user-attachments/assets/c9ae2eba-06f0-47da-99f2-b353a50a441e)

继续rebase合并server的操作，如下：

![image](https://github.com/user-attachments/assets/ff0a0a75-a88d-4bc5-beac-6dc6f3e9317c)

master合并server和client操作，然后删除这两个分支，如下：
![image](https://github.com/user-attachments/assets/bb15c46d-b51e-4d53-9a7f-fba07c1fea46)

如果不会git rebase --onto命令，使用diff/apply其实也可以实现上面的效果。

比如在C9开一个新分支，reset到C3，获取C8,C9的patch，打到C6上，然后提交，checkout到master分支，reset到C2，获取C3/4/5的patch，然后打到master分支上。
不过这种操作最大的问题是：丢失掉了分支的提交记录，并且繁琐。

假如有：A -> B -> C -> D -> E 这样的BR1提交记录，想把C和D合并到BR0分支，如何通过rebase实现？

可以以B为点，创建临时分支BR2，然后以D为点，创建临时分支BR3，在BR2分支上提交一个空提交`git commit --allow-empty -m "temp empty commit"`
在BR3分支上，执行`git rebase --onto BR0 BR2 BR3`，即可实现将BR1上的C和D合并到BR0分支上。

感觉也挺麻烦的，需要额外的创建两个分支，不过分支的提交记录可以得到保留。

绝对不要执行的rebase红线操作：永远不要对已经合入远端仓库的提交执行rebase变基操作，那会让事情一团糟。（作者的口吻很严厉）

比如：远端仓库有两个分支，你将其中一个分支的修改合并到另一个分支（采用merge方法），并且已经远端库完成修改合入。
             此时如果你出于想让提交记录变得更加清晰简洁，重新执行了rebase变基操作，然后重新push --force提交，覆盖掉原来的提交记录。
             如果有其他开发者，已经pull了之前采用merge方法合并的代码，再次提交后会变的混乱（提交记录）.


之所以不建议这样用rebase，是因为每次它执行之后，都不可避免的丢失掉一些提交信息，比如上面这些图中的灰色提交节点。
rebase丢弃掉旧的提交节点，创建新的提交节点，然而其他开发者可能还在旧的节点基础上进行开发，一旦rebase合入主线，
那么本地的其他开发者就会变得很尴尬：在一个不存在（已丢弃）的节点上进行开发。

如果这种情况已经发生了，作者也给出了解决建议，不过我想直接在团队禁止这样做，或者执行revert操作，是更好的方法

到底是采用merge还是rebase合并分支，取决于对提交记录的重视程度，如果不允许丢失任何记录，则使用merge，如何不太在意，可以使用rebase.


反馈池

+ .git/index这个二进制文件下保存了什么内容？
	- blob对象的哈希值、跟踪文件的状态（大小、修改时间、权限等等信息）、文件状态标记（新增/修改/删除）、冲突标记（冲突是否解决）等等信息


+ 为什么工作中提交的使用的是`HEAD:refs/for/master`而不是`HEAD:refs/heads/master`？？这两个有啥区别？
	- refs/for/master格式的分支引用，是专门用于代码审查的特殊分支引用格式，在gerrit完成代码审查后，会合并到refs/heads/matser中.
	- 如果没有代码审查的话(比如私人的远端代码服务器)，可以使用标准的`refs/heads/master`.

+ 为什么当HEAD指向commit-id，处于no-branch状态时，依然可以提交代码，此时git是如何知道使用本地的那个分支来更新远端分支的呢？
	- chatGPT的回答是，因为你使用的命令是`git push origin HEAD:refs/for/master`，其中refs/for/matser是特殊的gerrit审查分支引用，
	即使处于no-branch状态依然可以提交，因为此时其实你并没有提交代码到真正的远端仓库
	gerrit会为这个提交创建特殊的分支，当团队成员合入该笔提交后，会将该特殊分支的变更，更新到真正的远端仓库
	所以，在看不见的地方，git和gerrit默默做了很多事情

+ 如果本地不是no-branch状态，gerrit还会给提交，创建特殊分支吗？
	- 无论是不是出于no-branch状态，gerrit都会创建专门分支.


+ `git push origin --delete serverfix`本地删除远程分支吗？






























