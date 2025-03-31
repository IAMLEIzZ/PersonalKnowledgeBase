1. 创建仓库：在某个文件夹下 git init
	1. 会在当前目录下创建一个.git 目录，以存储信息
2. 查看当前仓库状态 git status
3. 将文件添加到暂存区 git add
	1. git add *.txt (通配符添加)
	2. git add . (添加当前文件夹下所有文件)
4. 把暂存区文件提交到仓库 git commit -m + "提交信息"
5. 回退版本 git reset + soft/hard/mixed
6. 回溯操作 git reflog
7. 查看文件差异 git diff(默认对比的是工作区和暂存区)，后面加两个版本提交 id 可以查看两个版本的差异
8. git rm 删除工作区和暂存区对应的文件