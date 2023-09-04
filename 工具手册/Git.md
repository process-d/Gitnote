**git基本命令**
git init 将本地文件夹初始化为git仓库
git clone 项目url  将项目下载到本地
项目url分为：https和git
git add: 将修改的文件提交到缓存区，在idea中文件变成绿色，没加入缓存区的文件是红色
git add .：将工作区所有变化的文件添加到暂存区，包括新添加的文件、被修改的文件和被删除的文件。
git commit -m "提交说明"：提交到本地仓库。此时文件会变成蓝色
git push 推送代码到远程仓库
git push github util：推送到github远程特定分支util,推送失败报连接错误可以加个-u参数
git push -u ：建立本地分支和远程分支的连接。
git pull：同步远程代码到本地
**git分支相关命令**
git checkout -b dev/1.5.4 origin/dev/1.5.4  从远程dev/1.5.4分支取到本地分支/dev/1.5.4
git checkout master 取出master版本的head，切换到master分支
git checkout util  表示从远程的util分支取出到本地分支util
git checkout . ：当前目录所有修改的文件 从HEAD中签出并且把它恢复成未修改时的样子
git checkout -：返回上一个分支
git branch 名称  创建分支
git branch 查看分支
**git一些查看命令**
git log：查看所有提交的日志
git status：查看当前分支做了哪些修改
git config --list 查看配置信息
cat .git/config 查看配置信息
git remote add git地址 添加远程仓库
git rm file删除文件
git 撤销更改：工作区->git add->暂存区->git commit->版本库。
还没有git add .就是没有加入到暂存区
用git status查看更改的文件，git checkout -- file撤销
git checkout .撤销工作区中所有修改
已经采用git add .加入到了暂存区
采用git reset HEAD file撤销或者采用git rm --cached 文件名
采用了git commit提交到了版本库中，
采用git reset HEAD file撤销

git remote add origin [https://github.com/process-d/Gitnote.git](https://github.com/process-d/Gitnote.git)
将本地仓库与远程仓库关联
# Git clone 私人项目 git clone [http://tokens（生成的token）@github.com/YOUR-USERNAME/YOUR-REPOSITORY（github地址）](http://tokens（生成的token）@github.com/YOUR-USERNAME/YOUR-REPOSITORY（github地址）)
token生成：在github的setting中的developer setting中设置中找到生成token
提交不上github的解决方法：在确定有网的情况下，
git config --global --unset http.proxy
git config --global --unset https.proxy
可以不用这个git config --global https.proxy
再提交
github已经禁止使用用户名和密码的方式连接github了，可以使用ssh进行连接
先在本地生成ssh密钥，然后将公钥复制到github中，之后需要自己的连接为git@而不是https:,可以用git remote -v查看，git remote set-url origin git@github.com:process-d/Gitnote.git 用来设置。