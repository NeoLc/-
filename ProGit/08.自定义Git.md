Git配置

git的配置文件有三类：
	•  /etc/config：对所有用户有效
	•  ~/.gitconfig：对当前用户所有仓有效
	• .git/config：对当前仓有效



常见的，比较有用的git配置：

	• commit.template

该参数可以用于设置统一的提交格式，用于规范团队的提交记录。

用法：git config --global commit.template ~/.gitmessge.txt

~.gitmessage.txt中可以自定义提交的格式.


	• core.excludesfile

忽略文件的方法好几种
+ 添加 .gitignore 进行文件忽略 - 需提交到仓
+ 修改 .git/info/exclude 进行文件忽略 - 本地忽略
+ `git config --global core.excludesfile ~/.gitignore_global` - 设置所有的仓库都忽略的文件


	• core.autocrlf

该配置可以解决window/linux/mac跨平台协作过程中的格式CRLF冲突问题（行尾符回车CR，换行LF）

git config --global core.autocrlf true - 自动完成不同平台行尾格式转换

	• core.whitespace

检测和修正多余空白字符的问题


MISC

git钩子，钩子是指当执行git特定操作时，会自动触发脚本。

钩子脚本放置在：.git/hook目录下, 以sample后缀结尾，如下图所示：

![image](https://github.com/user-attachments/assets/136dc3aa-bbf3-4dda-ae35-a91f5a9ae6b0)


当想触发某个钩子时，将sample后缀删除掉即可，例如上图中的commit-msg

不同名字的钩子，表示不同的时机流程，例如：

+ pre-commit ：提交信息前运行
+ commit-msg：用来在提交通过前验证项目状态或提交信息
+ pre-rebase 钩子运行于变基之前

想要绕过钩子，执行git命令时，添加`--no-verify`即可
