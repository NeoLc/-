## misc

`git clone`命令会在目录下创建.git目录，远端仓库下载的数据都会保存在该目录下，然后从.git中拷贝出最新版本的内容(变为工作区)

`git status -s` 显示简洁的输出，-s(short)，不加-s会显示详细的内容。

`.gitignore`的部分简单语法

![image](https://github.com/user-attachments/assets/749dcf36-75fc-43f0-8c5d-ab75dfb2bfa6)

`git diff` 显示尚未暂存的修改，换个角度说，就是查看即将进入暂存区的修改.

git diff --staged/--cached 显示即将从暂存区进入到版本库中的修改.

git log --stat 中--stat会显示统计信息，这是一个挺不错的命令，可以显示修改了那些文件，修改量多大，要比经常用的--name-only好用.

git log --pretty=fomat 挺强大的命令，可能用的不多，但是最好记住。

![image](https://github.com/user-attachments/assets/fdb3fcc4-0f42-4eea-a484-b3be465a8cf2)

git remote 命令可以查看远端仓库的名字
git remote show 命令可以查看远端仓库的详细信息
git remote -v 该命令可以查看远端仓库的url
git clone 命令执行后，会自动将其添加为远端仓库，并默认以“origin”命令

git fetch remote_name 会从远端仓库拉去最新内容到本地版本库中，注意此时本地版本库会发生更新，但是工作区并没有发生更新，需要再执行合并操作更新工作区。
                      fecth操作从远端分支拉取新的commit/tree/blob对象下载到本地的objects目录下，同时还会更新历史提交记录等内容.

git merge  执行fetch之后再执行merge操作，假设无冲突的情况下，git会将版本库中的最新改动覆盖工作区，并更新暂存区，最后工作区、暂存区、版本库内容保持一致。

git pull 操作会自动执行git fetch + git merge ，更加简洁方便。

git tag -a tag_name commit_id 可以对过去的提交打标签，PS：默认情况下本地打的标签并不会被git push提交到服务器，如果想共享标签的话可以通过git push origin --tags实现标签提交。

## 别名

git中有很多命令都很长很繁琐，功能虽然不错，但是写起来太麻烦了，此时可以采用别名操作。

例如：git log --stat，通过如下操作设置别名：

git config alias.lst 'log --stat'

此时执行 git lst 效果等价于`git log --stat`

别名功能还支持一个比较有趣(也比较少用)的功能，就是支持对给非原生git命令起别名，一般用于将Git仓储处理相关的第三方工具，自定义为git命令的场景。

比如linux原生的find命令，通过下面的操作，将其设置为git的新命令：git config alias.f '!find'，此时执行git f，效果和执行find一模一样。
和之前配置别名的区在于需要在第三方命令之前加一个”！符号“。

