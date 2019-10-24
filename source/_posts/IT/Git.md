---
title: Git
categories: 工具
---


## 一、Git 工作流程

<center>
![](http://dzliving.com/Git_0.png)
</center>

- Workspace：工作区
- Index / Stage：暂存区（和 git stash 命令暂存的地方不一样）
- Repository：仓库区（或本地仓库）
- Remote：远程仓库

#### 1.1 工作区

程序员进行开发改动的地方，是你当前看到的，也是最新的。

平常我们开发就是拷贝远程仓库中的一个分支，基于该分支进行开发。在开发过程中就是对工作区的操作。

#### 1.2 暂存区

.git 目录下的 index 文件，暂存区会记录 `git add` 添加文件的相关信息(文件名、大小、timestamp，...)，不保存文件实体，通过 `id` 指向每个文件实体。可以使用 `git status` 查看暂存区的状态。暂存区标记了你当前工作区中，哪些内容是被 git 管理的。

当你完成某个需求或功能后需要提交到远程仓库，那么第一步就是通过 git add 先提交到暂存区，被 git 管理。

#### 1.3 本地仓库

保存了对象被提交过的各个版本，比起工作区和暂存区的内容，它要<font color=#cc0000>更旧</font>一些。

`git commit` 后同步 index 的目录树到本地仓库，方便从下一步通过 git push 同步本地仓库与远程仓库的同步。

#### 1.4 远程仓库

远程仓库的内容可能被分布在多个地点的处于协作关系的本地仓库修改，因此它可能与本地仓库同步，也可能不同步，但是它的内容是最旧的。

#### 1.5 小结

- 任何对象都是在工作区中诞生和被修改；
- 任何修改都是从进入 index 区才开始被版本控制；
- 只有把修改提交到本地仓库，该修改才能在仓库中留下痕迹；
- 与协作者分享本地的修改，可以把它们push到远程仓库来共享。

下面这幅图更加直接阐述了四个区域之间的关系，可能有些命令不太清楚，没关系，下部分会详细介绍。

<center>
![](http://dzliving.com/Git_1.png)
</center>


## 二、常用的 Git 命令

<center>
![](http://dzliving.com/Git_2.png)
</center>

#### 2.1 HEAD

<center>
![](http://dzliving.com/Git_HEAD.png)
</center>

HEAD 始终指向当前所处分支的最新的提交点。你所处的分支变化了，或者产生了新的提交点，HEAD 就会跟着改变。

#### 2.2 add

<center>
![](http://dzliving.com/Git_Add.png)
</center>

add 主要实现将工作区修改的内容提交到暂存区，交由 git 管理。

<center>

|命令|描述|
|:----|:-----|
|git add .|添加当前目录的所有文件到暂存区|
|git add \<dir\>|添加指定目录到暂存区，包括子目录|
|git add \<file1\>|添加指定文件到暂存区|

</center>

#### 2.3 commit

<center>
![](http://dzliving.com/Git_Commit.png)
</center>

commit 主要实现将暂存区的内容提交到本地仓库，并使得当前分支的 HEAD 向后移动一个提交点。

<center>

|命令|描述|
|:----------|:---------|
|git commit -m \<message\>|提交暂存区到本地仓库，message 代表说明信息|
|git commit <file1> -m \<message\>|提交暂存区的指定文件到本地仓库|
|git commit --amend -m \<message\>|使用一次新的 commit，替代上一次提交|

</center>


#### 2.4 branch

<center>
![](http://dzliving.com/Git_Branch.png)
</center>

涉及到协作，自然会涉及到分支。关于分支，大概有四种操作：

1. 展示分支
2. 切换分支
3. 创建分支
4. 删除分支

分支是用来标记特定代码的提交，每一个分支通过 <font color=#cc0000>`SHA1sum`</font> 值来标识，所以对分支的操作是轻量级的，你改变的仅仅是 SHA1sum 值。

<center>

|命令|描述|
|:--------|:---------|
|git branch|列出所有本地分支|
|git branch -r|列出所有远程分支|
|git branch -a|列出所有本地分支和远程分支|
|git branch \<branch-name\>|新建一个分支，但依然停留在当前分支|
|git checkout -b \<branch-name\>|新建一个分支，并切换到该分支|
|git branch --track \<branch\>\<remote-branch\>|新建一个分支，与指定的远程分支建立追踪关系|
|git checkout \<branch-name\>|切换到指定分支，并更新工作区|
|git branch -d \<branch-name\>|删除分支|
|git push origin --delete \<branch-name\>|删除远程分支|

</center>

#### 2.5 fetch

|命令|描述|
|:------|:------|
|git fetch <远程主机名>|将某个远程主机的更新全部取回本地|
|git fetch <远程主机名> <分支名>|取回特定分支的更新|
|git fetch origin master|取回 origin 主机的 master 分支|

取回更新后，会返回一个 FETCH_HEAD，指的是某个 branch 在服务器上的最新状态，我们可以在本地通过它查看刚取回的更新信息：

```
$ git log -p FETCH_HEAD
```


#### 2.6 merge

<center>
![](http://dzliving.com/Git_Merge.png)
</center>

merge 命令把不同的分支合并起来。如上图，在实际开放中，我们可能从 master 分支中切出一个分支，然后进行开发完成需求，中间经过 R3、R4、R5 的 commit 记录，最后开发完成需要合入 master 中，这便用到了 merge。

<center>

|命令|描述|
|:--------|:-------|
|git fetch \<remote\>|merge 之前先拉一下远程仓库最新代码|
|git merge \<branch\>|合并指定分支到当前分支|

</center>

一般在 merge 之后，会出现 conflict，需要针对冲突情况，手动解除冲突。主要是因为两个用户修改了同一文件的同一块区域。

1. 中断合并

	```
	git merge --abort
	```
	
2. 撤销合并
	
	撤销合并时采用 git reset/revert 操作。

#### 2.7 rebase

<center>
![](http://dzliving.com/Git_Rebase.png)
</center>

rebase 又称为衍合，是合并的另外一种选择。

在开始阶段，我们处于 new 分支上，执行 `git rebase dev`，那么 new 分支上新的 commit 都在master 分支上重演一遍，最后 checkout 切换回到 new 分支。这一点与 merge 是一样的，合并前后所处的分支并没有改变。

git rebase dev，通俗的解释就是 new 分支想站在 dev 的肩膀上继续下去。rebase也需要手动解决冲突。

1. rebase 与 merge 的区别

	现在我们有这样的两个分支：test 和 master，提交如下：
	
	```
	      D---E test
	     /
	A---B---C---F master
	```
	
	在 master 执行 git merge test，然后会得到如下结果：
	
	```
	      D--------E
	     /          \
	A---B---C---F----G   test, master
	```
	
	在 master 执行 git rebase test，然后得到如下结果：
	
	```
	A---B---D---E---C'---F'   test, master
	```
	
	可以看到，merge 操作会<font color=#cc0000>生成一个新的节点</font>，之前的提交分开显示。而 rebase 操作<font color=#cc0000>不会生成新的节点</font>，是将两个分支融合成一个线性的提交。
	
如果你想要一个干净的，没有 merge commit 的线性历史树，那么你应该选择 git rebase；如果你想保留完整的历史记录，并且想要避免重写 commit history 的风险，你应该选择使用 git merge。

#### 2.8 reset

<center>
![](http://dzliving.com/Git_Reset.png)
</center>

reset 命令把当前分支指向另一个位置，并且相应的变动工作区和暂存区。

<center>

|命令|描述|
|:------|:-------|
|git reset —soft \<commit\>|只改变提交点，暂存区和工作目录的内容都不改变|
|git reset —mixed \<commit\>|改变提交点，同时改变暂存区的内容|
|git reset —hard \<commit\>|暂存区、工作区的内容都会被修改到与提交点完全一致的状态|
|git reset --hard HEAD|让工作区回到上次提交时的状态|

</center>


#### 2.9 revert

<center>
![](http://dzliving.com/Git_Revert.jpg)
</center>


git revert 用一个新提交来消除一个历史提交所做的任何修改。

1. revert 与 reset 的区别

	<center>
	![](http://dzliving.com/Git_ResetVSRevert.jpg)
	</center>
	
	- git revert 是用一次新的 commit 来回滚之前的 commit，git reset 是直接删除指定的commit。
	- 在回滚这一操作上看，效果差不多。但是在日后继续 merge 以前的老版本时有区别。因为 git revert 是用一次逆向的 commit“中和”之前的提交，因此日后合并老的 branch 时，导致这部分改变不会再次出现，减少冲突。但是 git reset 是之间把某些 commit 在某个 branch 上删除，因而和老的branch 再次 merge 时，这些被回滚的 commit 应该还会被引入，产生很多冲突。关于这一点，不太理解的可以看[这篇文章](https://link.jianshu.com/?t=http://yijiebuyi.com/blog/8f985d539566d0bf3b804df6be4e0c90.html)。
	- git reset 是把 HEAD 向后移动了一下，而 git revert 是 HEAD 继续前进，只是新的commit 的内容和要 revert 的内容正好相反，能够抵消要被 revert 的内容。


#### 2.10 push

上传本地仓库分支到远程仓库分支，实现同步。

<center>

|命令|描述|
|:--------|:-------|
|git push \<remote\>\<branch\>|上传本地指定分支到远程仓库|
|git push \<remote\> --force|强行推送当前分支到远程仓库，即使有冲突|
|git push \<remote\> --all|推送所有分支到远程仓库|

</center>

#### 2.11 pull

git pull 的过程可以理解为：

```
git fetch origin master   // 从远程主机的 master 分支拉取最新内容 
git merge FETCH_HEAD      // 将拉取下来的最新内容合并到当前所在的分支中
```

即将远程主机的某个分支的更新取回，并与本地指定的分支合并，完整格式可表示为：

```
$ git pull <远程主机名> <远程分支名>:<本地分支名>
```

如果远程分支是与当前分支合并，则冒号后面的部分可以省略：

```
$ git pull origin next
```


#### 2.12 stash

|命令|描述|
|:------|:------|
|git stash save "xx"|执行存储，并添加备注，只执行 git stash 也是可以的，但查找时不方便识别|
|git stash list|查看 stash 了哪些存储|
|git stash show|显示做了哪些修改，默认 show 第一个存储，如果要显示其他存储，后面加stash@{$num}，比如第<font color=#cc0000>二</font>个 `git stash show stash@{1}`|
|git stash show -p|显示第一个存储的改动，如果想显示其他存储，加上 stash@{$num}，比如第二个：`git stash show  stash@{1}  -p`|
|git stash apply|应用某个存储，但不会把存储从存储列表中删除，默认使用第一个存储，即 stash@{0}，如果要使用其他的，git stash apply stash@{$num} ， 比如第二个：git stash apply stash@{1} |
|git stash pop|命令恢复之前缓存的工作目录，将缓存堆栈中的对应 stash 删除，并将对应修改应用到当前的工作目录下，默认为第一个 stash，即 stash@{0}，如果要应用并删除其他 stash，命令：git stash pop stash@{$num}，比如应用并删除第二个：git stash pop stash@{1}|
|git stash drop stash@{$num}|丢弃 stash@{$num} 存储，从列表中删除这个存储|
|git stash clear|删除所有缓存的stash|

<center>
![](http://dzliving.com/Git_Stash.png)
</center>

> **注意**：没有在 git 版本控制中的文件，是不能被 git stash 存起来的。

如果新增了一个文件，直接执行 git stash 是不会存起来的，可以先执行 git add，再执行 git stash。

这个时候，想切分支就再也不会报错有改动未提交了。

如果要应用这些 stash，直接使用 git stash apply 或者 git stash pop 就可以再次导出来了。

1. 总结 
	
	git add 只是把文件加到 git 版本控制里，并不等于就被 stash 起来了，git add 和 git stash 没有必然的关系，但是执行 git stash 能正确存储的前提是文件必须在 git 版本控制中才行。

	常规 git stash 的一个限制是它会一下暂存所有的文件。有时，只备份某些文件更为方便，让另外一些与代码库保持一致。一个非常有用的技巧，用来备份部分文件：
	
	* add 那些你不想备份的文件
	* 调用 git stash –keep-index。只会备份那些没有被 add 的文件。
	* 调用 git reset 取消已经 add 的文件的备份，继续自己的工作。

#### 2.13 其他命令

<center>

|命令|描述|
|:------|:-------|
|git status|显示有变更的文件|
|git log|显示当前分支的版本历史|
|git diff|显示暂存区和工作区的差异|
|git diff HEAD|显示工作区与当前分支最新 commit 之间的差异|
|git cherry-pick \<commit\>|选择一个 commit，合并进当前分支|

</center>

## 三、Git Reset 三种模式

使用 Git 时有可能 commit 提交代码后，发现这一次 commit 的内容是有错误的，那么有两种处理方法：

1. 修改错误内容，再次 commit一次
2. 使用 git reset 命令撤销这一次错误的 commit

第一种方法多一条 commit 记录；第二种方法，错误的 commit 不会被保留下来。

> git reset：Reset current HEAD to the specified state。让 HEAD 指针指向其他的地方。

例如我们有一次 commit 不是很满意，需要回到上一次的 Commit 里面。那么这个时候就需要通过 reset，把 HEAD 指针指向上一次的 commit 的点。

它有三种模式：soft、mixed、hard。

<center>
![git各个区域和命令关系](http://dzliving.com/Git_Reset_0.png)
</center>

这三个模式理解了，对于使用这个命令很有帮助。在理解这三个模式之前，需要略微知道一点 Git 的基本流程。

<center>
![](http://dzliving.com/Git_Reset_1.png)
</center>

简单叙述一下把文件存入 Repository 流程：

1. 刚开始 working tree、index 与 repository(HEAD) 里面的内容都是一致的。

	<center>
	![](http://dzliving.com/Git_Reset_2.png)
	</center>

2. 当 git 管理的文件夹里面的内容出现改动后，此时 working tree 的内容就会跟 index 及 repository(HEAD) 的不一致，而 Git 知道是哪些文件(Tracked File)被改动过，直接将文件状态设置为 modified (Unstaged files)。
	
	<center>
	![](http://dzliving.com/Git_Reset_3.png)
	</center>

3. 当我们执行 `git add` 后，会将这些改变的文件内容加入 index 中 (Staged files)，所以此时working tree 跟 index 的内容是一致的，但与 repository(HEAD) 内容不一致。

	<center>
	![](http://dzliving.com/Git_Reset_4.png)
	</center>

4. 接着执行 `git commit` 后，将 Git 索引中所有改变的文件内容提交至 Repository 中，建立出新的 commit 节点(HEAD)后， working tree、index 与 repository(HEAD) 区域的内容 又会保持一致。

	<center>
	![](http://dzliving.com/Git_Reset_5.png)
	</center>


#### 3.1 reset --hard

> reset --hard 会在重置 HEAD 和 branch 的同时，重置 stage 区和工作目录里的内容。

当你在 reset 后面加了 --hard 参数时，你的 stage 区和工作目录里的内容会被完全重置为和 HEAD 的新位置相同的内容。换句话说，就是你的没有 commit 的修改会被全部擦掉。

<center>
![git status](http://dzliving.com/Git_Reset_6.png)
</center>

然后，你执行了 reset 并附上了 --hard 参数：

```
git reset --hard
```

你的 HEAD 和当前 branch 切到上一条 commit 的同时，你工作目录里的新改动和已经 add 到 stage 区的新改动也一起全都消失了：

<center>
![git status](http://dzliving.com/Git_Reset_7.png)
</center>

可以看到，在 reset --hard 后，所有的改动都被擦掉了。


#### 3.2 reset --soft

> reset --soft 会在重置 HEAD 和 branch 时，保留工作目录和暂存区中的内容，并把重置 HEAD 所带来的新的差异放进暂存区。

什么是「重置 HEAD 所带来的新的差异」？就是这里：

<center>
![](http://dzliving.com/Git_Reset_Soft.gif)
</center>

由于 HEAD 从 4 移动到了 3，而且在 reset 的过程中工作目录和暂存区的内容没有被清理掉，所以 4 中的改动在 reset 后就也成了工作目录新增的「工作目录和 HEAD 的差异」。这就是上面一段中所说的「重置 HEAD 所带来的差异」。

此模式下会保留 working tree 工作目录的内容，不会改变到目前所有的 git 管理的文件夹的内容；也会保留 index 暂存区的内容，让 index 暂存区与 working tree 工作目录的内容是一致的。就只有 repository 中的内容的更变需要与 reset 目标节点一致，因此原始节点与 reset 节点之间的差异变更集合会存在与 index 暂存区中(Staged files)，所以我们可以直接执行 git commit 将 index 暂存区中的内容提交至 repository 中。当我们想合并「当前节点」与「reset 目标节点」之间不具太大意义的 commit 记录（可能是阶段性地频繁提交）时，可以考虑使用 Soft Reset 来让 commit 演进线图较为清晰点。

1. 修改后的 AppDelegate.h 文件 add 到 stage 区，修改后的 AppDelegate.m 保留在工作目录

	<center>
	![](http://dzliving.com/Git_Reset_8.png)
	</center>

2. 查看当前最新的 commit 记录

	```
	git show --stat
	```
	
	<center>
	![](http://dzliving.com/Git_Reset_9.png)
	</center>
	

3. 执行 git reset --soft HEAD^

	<center>
	![](http://dzliving.com/Git_Reset_10.png)
	</center>
		
	那么除了 HEAD 和它所指向的 branch1 被移动到 HEAD^ 之外，原先 HEAD 处 commit 的改动（README.md 文件）也会被放进暂存区。
	
这就是 --soft 和 --hard 的区别：--hard 会清空工作目录和暂存区的改动，而 --soft 则会保留工作目录的内容，并把因为保留工作目录内容所带来的新的文件差异放进暂存区。


#### 3.3 reset --mixed

> reset 如果不加参数，那么默认使用 --mixed 参数。它的行为是：保留工作目录，并且清空暂存区。也就是说，工作目录的修改、暂存区的内容以及由 reset 所导致的新的文件差异，都会被放进工作目录。简而言之，就是「把所有差异都混合（mixed）放在工作目录中」。

以上面的情况为例：

<center>
![](http://dzliving.com/Git_Reset_11.png)
</center>

工作目录的内容和 --soft 一样会被保留，但和 --soft 的区别在于，它会把<font color=#cc0000>暂存区清空</font>，并把原节点和 reset 节点的差异的文件放在工作目录。总而言之就是，工作目录的修改、暂存区的内容以及由 reset 所导致的新的文件差异，<font color=#cc0000>都会被放进工作目录</font>。


#### 3.4 总结

> reset 的本质：移动 HEAD 以及它所指向的 branch。

实质上，reset 这个指令虽然可以用来撤销 commit，但它的实质行为并不是撤销，而是移动 HEAD ，并且「捎带」上 HEAD 所指向的 branch（如果有的话）。也就是说，reset 这个指令的行为其实和它的字面意思“重置”十分相符：它是用来重置 HEAD 以及它所指向的 branch 的位置的。

而 reset --hard HEAD^ 之所以起到了撤销 commit 的效果，是因为它把 HEAD 和它所指向的 branch 一起移动到了当前 commit 的父 commit 上，从而起到了「撤销」的效果：

<center>
![](http://dzliving.com/Git_Reset_Hard_HEAD.gif)
</center>

Git 的历史只能往回看，不能向未来看，所以把 HEAD 和 branch 往回移动，就能起到撤回 commit 的效果。

所以同理，reset --hard 不仅可以撤销提交，还可以用来把 HEAD 和 branch 移动到其他的任何地方。

```
git reset --hard branch2
```

<center>
![](http://dzliving.com/Git_Reset_Branch.gif)
</center>


#### 3.5 reset 三种模式区别和使用场景

1. 区别

	* --hard：重置位置的同时，直接将 working Tree工作目录、index 暂存区及 repository 都重置成目标 Reset 节点的内容，所以效果看起来等同于清空暂存区和工作区。
	* --soft：重置位置的同时，保留 working Tree 工作目录和 index 暂存区的内容，只让 repository 中的内容和 reset 目标节点保持一致，因此原节点和 reset 节点之间的【差异变更集】会放入 index 暂存区中(Staged files)。所以效果看起来就是工作目录的内容不变，暂存区原有的内容也不变，只是原节点和 Reset 节点之间的所有差异都会放到暂存区中。
	* --mixed（默认）：重置位置的同时，只保留 Working Tree 工作目录的内容，但会将 Index 暂存区和 Repository 中的内容更改和 reset 目标节点一致，因此原节点和 Reset 节点之间的【差异变更集】会放入 Working Tree 工作目录中。所以效果看起来就是原节点和 Reset 节点之间的所有差异都会放到工作目录中。

2. 使用场景

	* --hard：
		* 要放弃目前本地的所有改变时，即去掉所有 add 到暂存区的文件和工作区的文件，可以执行 git reset -hard HEAD 来强制恢复 git 管理的文件夹的内容及状态；
		* 真的想抛弃目标节点后的所有 commit（可能觉得目标节点到原节点之间的 commit 提交都是错了，之前所有的 commit 有问题）。
	* --soft
		* 原节点和 reset 节点之间的【差异变更集】会放入 index 暂存区中(Staged files)，所以假如我们之前工作目录没有改过任何文件，也没 add 到暂存区，那么使用 reset --soft 后，我们可以直接执行 git commit 将 index 暂存区中的内容提交至 repository 中。为什么要这样呢？这样做的使用场景是：假如我们想合并「当前节点」与「reset 目标节点」之间不具太大意义的 commit 记录(可能是阶段性地频繁提交,就是开发一个功能的时候，改或者增加一个文件的时候就commit，这样做导致一个完整的功能可能会好多个commit点，这时假如你需要把这些commit整合成一个commit的时候)时，可以考虑使用reset --soft来让 commit 演进线图较为清晰。总而言之，可以使用 --soft 合并 commit 节点。
	* --mixed（默认）
		* 使用完reset --mixed 后，我們可以直接执行 git add 将這些改变果的文件内容加入 index 暂存区中，再执行 git commit 将 Index暂存区 中的内容提交至Repository中，这样一样可以达到合并commit节点的效果（与上面--soft合并commit节点差不多，只是多了git add添加到暂存区的操作）；
		* 移除所有Index暂存区中准备要提交的文件(Staged files)，我们可以执行 git reset HEAD 来 Unstage 所有已列入 Index暂存区 的待提交的文件。(有时候发现add错文件到暂存区，就可以使用命令)。
		* commit提交某些错误代码，或者没有必要的文件也被commit上去，不想再修改错误再commit（因为会留下一个错误commit点），可以回退到正确的commit点上，然后所有原节点和reset节点之间差异会返回工作目录，假如有个没必要的文件的话就可以直接删除了，再 commit 上去就 OK 了。

## 四、文章

[Git](https://git-scm.com/docs)
[Ruheng](https://www.jianshu.com/u/0fa6f5d09040) - [一篇文章，教你学会Git](https://www.jianshu.com/p/072587b47515)
[carway](https://www.jianshu.com/u/772b2edab0f4) - [Git Reset 三种模式](https://www.jianshu.com/p/c2ec5f06cf1a)
[Git回滚Merge](https://www.jianshu.com/p/bb5b84c638f0)
[Git远程操作详解](http://www.ruanyifeng.com/blog/2014/06/git_remote.html)