#### 概念

##### 1. 结构

**工作区(Working Directory)**

平时编辑项目文件的地方，即用户能直接操作的目录，该目录下会有一个 `.git` 隐藏文件夹

**暂存区(Stage/Index)**

保存下次将要提交的文件列表信息，在 `.git` 中实际上是一个名为 `index` 的文件

**本地仓库(Git Repository)**

保存所有版本和相关信息的地方

**远程仓库(Remote Repository)**



##### 2. 状态

<img src="https://raw.githubusercontent.com/WeYan1223/Pic/master/Git/状态.png" alt="状态.png (800×330) (raw.githubusercontent.com)" style="zoom:80%;" /> 

**未追踪(Untracked)**

指未被纳入版本控制的文件，通常是在工作区添加的新文件

**未修改(Unmodified)**

表示工作目录的文件与本地仓库中当前版本的对应文件是一致的（未被修改）

**已修改(Modified)**

表示文件在工作区已被修改，但修改信息未添加到暂存区

**暂存(Staged)**

表示文件在工作区已被修改，且修改信息已添加到暂存区，但未提交到本地仓库

***

#### 简单用法

##### 1. 获取本地仓库

> 如下两种方式

**克隆远程仓库**

从服务器克隆本地仓库

````shell
git clone -b develop [url]
````

这里 `-b develop` 的作用是克隆 `develop` 分支（默认是 `master` 分支），克隆成功后所有文件都处于 `Unmodified` 状态

**初始化本地仓库**

对现有目录下的文件进行版本控制，需要在相应目录下执行如下命令

````shell
git init
````

此时原目录下会多了一个 `.git` 文件夹用于版本控制，原目录下所有文件都处于 `Untracked` 状态

##### 2. 添加文件

在工作目录中新添加的文件处于 `Untracked` 状态

````shell
git add [filename]
````

执行如上命令，新文件处于 `Stage` 状态

##### 3. 修改文件

修改一个已被纳入版本控制的文件，如 `README.md`，运行 `git status` 查看状态

````shell
$ git status
On branch master

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   README.md
````

此时文件处于 `Modified` 状态，执行如下命令将本次修改信息添加到暂存区

````java
git add README.md
````

执行如下命令将暂存区中的修改信息提交到本地仓库中，此时本地仓库中的当前分支会延申一个节点保存这次的提交信息，此时工作区中的所有文件的状态变为 `Unmodified`

````shell
git commit -m"提交说明"
````

##### 4. 删除文件

删除工作区中的 `README.md` 文件，查看状态

````shell
$ git status
On branch master

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        deleted:    README.md
````

后续操作与修改文件类似

##### 5. 撤销操作

* `Untracked` 状态的文件，由于未被纳入版本控制，所以 `Git` 并不能帮助此类文件进行撤销操作

* `Modified` 状态的文件，撤销操作的实际是丢弃工作区的修改，如下命令

  ````shell
  git restore [filename]
  ````

* `Staged` 状态的文件，撤销操作的实际是将暂存区的修改信息重新放回工作区，相当于撤销对该文件的 `git add` 命令，如下命令

  ````shell
  git restore --staged [filename]
  ````

* 如果文件的修改已经提交到本地仓库，撤销操作的实际是版本回退

##### 6. 版本回退

>  `HEAD` 可以理解为所在分支的指向当前节点的指针

`git reset` 命令用于版本回退，语法如下

````shell
git reset --[soft|mixed|hard] [版本] [文件]
````

示例：

````shell
$ git reset HEAD^            # 回退所有内容到上一个版本  
$ git reset HEAD^^ README.md # 回退README.md文件的版本到上上一个版本  
$ git reset 052e             # 通过提交id回退到指定版本
````

其中的提交 `id`，可以通过 `git log` 查看，这里还需要了解其中三个参数

* `--soft`：被回退的版本的**下一次**修改信息会保存在暂存区
* `--mixed`：默认选项，被回退的那版本的**下一次**修改信息会保存在工作目录
* `--hard`：被回退的版本的**下一次**修改信息会直接舍弃

##### 7. 分支管理

````shell
git branch [分支名]     # 创建分支
git switch [分支名]     # 切换分支
git branch -d [分支名]  # 删除分支
git branch -D [分支名]  # 强制删除分支
````

##### 8. 远程仓库

````shell
git remote add [远程仓库名] [url]                    # 添加远程仓库
git push [远程仓库名] [远程分支名]                     # 将当前分支推送到远程仓库（远程仓库不为空）
git push -u [远程仓库名] [远程分支名]                  # 将当前分支推送到远程仓库（远程仓库为空）
git checkout --track [本地分支名] [远程仓库名]/[分支名] # 跟踪远程分支
git fetch [远程仓库名] [远程分支名]                    # 将远程分支的的新修改拉取到本地，但不会合并到当前分支，需要手动合并
git pull [远程仓库名] [远程分支名]                     # 将远程分支的的新修改拉取到本地并合并到当前分支
git branch -u [远程仓库名] [远程分支名]                # 建立当前分支与远程分支的映射关系
git branch --set-upstream-to [远程仓库名] [远程分支名] # 建立当前分支与远程分支的映射关系
git branch --unset-upstream                        # 撤销本地分支与远程分支的映射关系
````

***

#### 分支合并

##### 1. 三向合并

> `Git` 中两个分支的合并需要使用三向合并

对于两个分支的合并，例如 `dev` 分支合并到 `master` 分支

1. 首先需要找到两个分支的一个公共节点 `base`
2. `dev` 分支与 `master` 分支的最新节点与 `base` 节点对比，对于同一处地方
   * `dev` 和 `master` 都没有修改，则不变
   * `dev` 或 `master` 其中一方做了修改，则保留该修改
   * `dev` 和 `master` 都做了修改，则发生冲突，需要手动解决冲突
3. 没有冲突或解决冲突后，`master` 会新增一个节点来记录这次合并

##### 2. 合并策略

> 常见的合并策略有 `Fast-forward`、`Recursive`、`Ours`、`Theirs`、`Octopus`，默认情况下 `Git` 会自动挑选合适的合并策略，如果要强制指定，使用命令
>
> ````shell
> git merge -s [合并策略] [分支名] # 将指定分支使用指定合并策略合并到当前分支
> ````

详情参考文章：[这才是真正的 Git——分支合并 | 机器之心](https://www.jiqizhixin.com/articles/2020-05-28-8)

##### 3. rebase

````shell
git rebase [分支名] # 将指定分支rebase到当前分支
````

详情参考文章：[git rebase详解](https://blog.csdn.net/weixin_42310154/article/details/119004977)

***

#### 远程仓库

##### 1. SSH

> 非对称加密协议：公钥加密、私钥解密

通常在 `~/.ssh` 目录下会存在如下三个文件：

* `id_rsa`：用户的私钥

* `id_rsa.pub`：用户的公钥

* `known_hosts`：保存 `Git` 服务器的公钥