```toc
```
git作为目前最主流的版本控制器，可以控制电脑中所有格式的文件
对于开发人员来说，通常使用git维护项目的源文件

那么什么是版本控制器？版本控制器是一个管理系统，用来记录每次的修改以及版本迭代（每次的版本数据）
## 基本操作
### 创建git仓库
要想使用git的版本控制，就需要创建一个git仓库，只有在仓库中的文件才会被git追踪管理
将当前所在目录初始化成git仓库：
```bash
git init
```

此时目录下会多出一个.git文件
![image.png](https://s2.loli.net/2023/11/09/mwAoHPVNYMOLWXG.png)

### 配置name和email
使用`git config`进行配置
```bash
git config user.name "用户名"
git config user.email "邮箱地址"
```
这两个配置项必须与git远程仓库对应
查看配置项
```bash
git config -l
```
![image.png](https://s2.loli.net/2023/11/09/AaiZoXkCvFH5GSu.png)

删除配置
```bash
git config --unset user.name
git config --unset user.email
```

使用`--global`选项选项，将配置应用于本地的所有git仓库：
```bash
git config --global user.name "用户名"
git config --global user.email "邮箱地址"
```

相应的，要删除全局的配置，只需要在删除命令中加上`--global`选项

## .git目录的结构
查看一下git的目录树
![image.png](https://s2.loli.net/2023/11/09/K7cEp8JHaICVDYi.png)
我们不能修改.git下的所有文件，也不能更改.git的目录结构，这将导致git出错

在.git所在目录下创建了文件，文件会受到git的版本控制吗？
![image.png](https://s2.loli.net/2023/11/09/BLwKkjpWxz2dgOM.png)
解释之前先说明两个概念：
1. 版本库（即仓库）：当前目录下的.git被称为版本库
2. 除了.git之外的其他文件被称为工作区。我们只能修改工作区中的内容，不能修改版本库中的内容

工作区与版本库的关系：
![image.png](https://s2.loli.net/2023/11/09/xSTZBWjbLNef3mO.png)
版本库中，stage被称为暂存区，也叫索引index。此外还有一个master分支，由`HEAD`指针指向

**工作区**中的修改（新增、修改、删除）不会受到git的追踪
执行add命令后，git会将工作区中的修改写入暂存区
执行commit后，git会将暂存区中的修改提交到master分支中，此时修改真正地被写入版本库，**这些修改才会受到git的追踪控制**

此外，版本库中还有一个分区：objects对象库。对象库中存在多个对象，所有对**工作区**的修改都会被写入到这些对象中（这些对象保存了工作区的修改）
可以注意到暂存区和master都是目录树的结构，其中存储了文件，而这些文件中存储了对象库中对象的指针。也就是说，暂存区和master通过保存对象库中的对象指针，以表示当前的状态

执行add将修改暂存区的目录树，使之索引到对象库中的对象
执行commit操也修改master的目录树，使之索引到对象库中的对象

## git add & git commit
将工作区中的修改写入暂存区中
```bash
git add filename [filename]
```
可以使用`git add .`，表示将当前目录下的所有修改写入暂存区中

将暂存区中的修改写入master分支（本地仓库）中
```bash
git commit -m "描述此次提交的目的"
```
使用`git commit`时，需要带上`-m`选项并输入此次提交的目的
如果直接输入`git commit`，我们将进入vim编辑器并输入此次提交的目的。显然直接使用`-m`选项是更方便的

打印时间从近到远的commit记录，记录包含了每次提交的提交目的
```bash
git log
```
![image.png](https://s2.loli.net/2023/11/09/K4wdi8NAhZrkgjY.png)

其中commit id是经过hash得到的定长字符串。对于`git log`命令，携带`--pretty=oneline`选项，将简化打印的内容为一行
![image.png](https://s2.loli.net/2023/11/09/LtRITnOfp9mzoG4.png)
### .git目录结构的变化
再创建两个文件，将其commit到仓库中
![image.png](https://s2.loli.net/2023/11/09/oyrCaLgVtjK9bHU.png)

此时观察.git的目录结构
![image.png](https://s2.loli.net/2023/11/09/65yzFafKu92lt1E.png)

其中新增了index文件，即暂存区。执行add操作后，暂存区保存了工作区的修改，所以此时出现了index文件

此外还有存在一个HEAD文件
cat一下HEAD文件，发现指向了仓库中的master文件，也cat一下master文件
![image.png](https://s2.loli.net/2023/11/09/J1h9ALMIVTCwZGO.png)
发现master文件保存了commit id，该commit id为最近一次commit操作的id

首先，master和index中保存着对象指针，指针指向的对象存储了工作区中的修改信息
所以commit id是一个指针，前两位为目录名，剩下的数字为文件名，这些文件存储在.git的objects目录下，可以在objects目录中找到对应文件
使用命令，打印object的信息
```bash
git cat-file -p commit id
```
`-p`选项可以将打印信息简化
![image.png](https://s2.loli.net/2023/11/09/mBGqncgDCwyiRHe.png)
（似乎不使用`-p`还无法打印信息）
可以看到object中记录了执行commit操作的用户名与邮箱，以及上一次的commit id
第一行还有一串commit id，git cat-file下，看看是什么
![image.png](https://s2.loli.net/2023/11/09/YgOUtG2MVBwkxuh.png)
可以看到，该对象似乎保存了工作区中每个文件的object指针，每个文件对应的object中保存了文件的修改信息
![image.png](https://s2.loli.net/2023/11/09/xgisa4HwD5ojMuN.png)
ReadMe文件中，确实只有"hello git"这串字符串

总结一下，object保存了每次的commit信息，以及工作区中每个文件的修改信息。如何获取object的指针？master文件保存了最近一次commit对应的object指针，也保存了上一次commit的object指针

注意：要想将工作区中的修改提交到版本库中，必须先执行add将修改提交到工作区，再执行commit将修改提交到版本库，这两个操作不能颠倒
## git追踪管理的数据
**git追踪管理的不是文件，而是每一次的修改**
修改ReadMe文件，使用`git status`命令查看当前仓库的状态
![image.png](https://s2.loli.net/2023/11/09/CDW6JmuyYx4IZOz.png)
打印信息说，我们修改了ReadMe文件，但是暂存区中没有文件（no changes added）需要被提交
因为我们没有执行add

使用以下命令，查看工作区与暂存区中\[某个文件\]的差异
```bash
git diff [filename]
```
![image.png](https://s2.loli.net/2023/11/09/kiHEZl3yCu9xaRw.png)
打印的是unix通用的diff格式：
`---`和`a`表示改动前，`+++`和`b`表示改动后
-1表示第一行为改动前的内容，+1表示之后的一行为改动后的内容，2表示改动后的内容到第二行结束

使用以下命令，查看版本库与工作区中\[某个文件\]的差异
```bash
git diff HEAD [filename]
```
	![image.png](https://s2.loli.net/2023/11/09/7H8IQFR6neid1Ck.png)

将当前工作区的修改add到暂存区中，执行`git status`
![image.png](https://s2.loli.net/2023/11/09/iCj8YPESlTcUGXF.png)
打印信息说，当前暂存区中的信息已经准备好被提交了（changes to be committed）
并且指明了暂存区中修改的文件为ReadMe

commit之后，再执行`git status`
![image.png](https://s2.loli.net/2023/11/09/o7QEIVjer8zZqbh.png)
打印信息说，工作区与暂存区都没有发生修改
### git的版本回退
版本回退本质上是回退**版本库**中的内容
使用以下命令，进行版本回退
```bash
git reset [--soft | --mixed | --hard] [HEAD] -- commit id
```
- --soft：回退版本库中的内容
- --mixed：回退版本库与暂存区中的内容，可以指定需要回退的文件
- --hard：回退所有区域的内容

其中miexed为默认选项
之前说过，执行了`git init`后的目录下有三个区域，工作区、暂存区以及版本库（仓库），reset命令的不同选项将不同程度地回退这些区域中的内容
注意：hard选项需要慎用！因为它将回退工作区中的数据！

根据`git log`打印的commit id，进行hard选项的版本回退，使ReadMe中只有"hello git"
![image.png](https://s2.loli.net/2023/11/09/2sFZKC9ypvVjYbd.png)
可以看到之前有的"appending..."消失，只剩下"hello git"，并且日志也进行了回退

如果你有回退之前的commit id，你甚至可以撤销回退，回退之前的回退
![image.png](https://s2.loli.net/2023/11/09/RTlsw3gzqOuPrMd.png)
该commit id为回退之前的id，此时成功撤销了回退
同时日志的回退也被撤销了
![image.png](https://s2.loli.net/2023/11/09/lWSTBbVjsKczy69.png)

如果想要撤销回退，执行`git reflog`可以查看所有add和commit的信息，其中就包含了commit id的前7个字符，使用commit id的前7个字符也能进行执行/撤销回退
![image.png](https://s2.loli.net/2023/11/09/ACxXn7zyBvKRZmQ.png)
而一旦找不到commit id，将无法执行/撤销回退的操作
#### 回退的原理
为什么git的回退这么快？因为回退只是修改了master保存的指针
master存储了最近一次的commit id，也就是object指针。object对象记录了每一次的修改操作。版本回退本质上是在修改master的值，使master指向不同的object指针。此时仓库中最近一次的commit操作就被修改，最近一次的commit不同，仓库中的数据就不同
#### 回退的三种情况
撤销操作的本质：防止错误的代码从版本库中被push到**远程仓库**中
用code表示错误的代码，那么将出现以下三种情况

|工作区|暂存区|版本库|解决方法|
|-|-|-|-|
|code|||git checkout -- |
|code|code||git reset + git checkout --或者git reset --hard|
|code|code|code|git reset --hard，前提条件: commit之后没有push|

1. 工作区中开发了大量代码，但是没有add，此时要撤销工作区中的修改
2. 工作区中开发了大量代码，add了但是没有commit，此时要撤销工作区与暂存区的修改
3. 工作区中开发了大量代码，add并且commit了，此时要撤销工作区、暂存区与版本库的修改

情况一：只有工作区需要回退代码

执行命令，将工作区的文件回退到最近一次add的状态，也就是和暂存区一致
```bash
git checkout -- filename
```
注意，必须要加上`--`，否则命令的效果完全不一样！
![image.png](https://s2.loli.net/2023/11/09/l6CX8wZMyBGFf7k.png)

情况二：工作区和暂存区都需要回退代码

先修改ReadMe中的内容，添加"code"并且add修改到暂存区中
![image.png](https://s2.loli.net/2023/11/10/lL2qwZ3rPdkTtMm.png)

两种解决方法：
执行hard回退命令，使工作区与暂存区中的代码和版本库一致
```bash
git reset --hard HEAD filename
```
![image.png](https://s2.loli.net/2023/11/10/6RKQOPG8uErjLgz.png)
注意，`git reset --hard`不能回退指定文件，只有`mixed`模式能回退指定文件！

另一种方法：将情况二转换成情况一
执行命令，使暂存区中的代码回退，与版本库中的代码一致
```bash
git reset HEAD filename
```
`HEAD`指针指向了master分支，而master分支保存版本库的最近一次修改，即最新版本
所以`git reset HEAD`将使**版本库和暂存区**回退到当前版本，而版本库本来就是当前版本，所以只有暂存区回退到了当前版本

用`git status`验证：
![image.png](https://s2.loli.net/2023/11/10/EVtqQMRZOlXchUG.png)
此时`git status`说，暂存区中没有修改需要被提交，也就是暂存区的内容和工作区中的内容一致

此时执行`git checkout -- ReadMe`，使工作区中版本和暂存区中的版本一致即可
![image.png](https://s2.loli.net/2023/11/10/x8hp4CZWmAyGrIH.png)

情况三：所有区域都需要回退代码

执行命令，将工作区、暂存区以及版本库中的代码回退到上一版本
```
git reset --hard HEAD^
```
（其中`HEAD`，表示上一次修改即**当前版本**，而`HEAD^`表示上一个版本，`HEAD^^`表示上两个版本...）
一般来说都是将代码回退到上一版本，但是具体来说是哪个版本，就需要根据情况决定了
![image.png](https://s2.loli.net/2023/11/10/DYECiwLrNJaIHlU.png)
## 版本库中文件的删除
第一种方式：
1. 先删除工作区中的文件
2. 再将修改提交到暂存区中
3. 最后将暂存区中的修改提交到版本库中

```bash
rm file1
git add file1
git commit -m "delete file1"
```
![image.png](https://s2.loli.net/2023/11/10/SzlQUm7d3XEinLx.png)
可以看到当前版本库中已经没有file1文件了

第二种方式：
1. 执行git rm，同时删除工作区与暂存区中对应文件
2. 将暂存区中的修改commit到版本库中

其实就是`git rm`等价于`rm+git add`
```bash
git rm file2
git commit -m "delete file2"
```
![image.png](https://s2.loli.net/2023/11/10/Lz3peHFMIgQYrnJ.png)
可以看到当前目录只剩下了ReadMe文件
## git分支管理
之前说过，`HEAD`指针指向master分支，master分支存储了一个对象指针，该对象保存了最近一次commit，查看该对象的数据
![image.png](https://s2.loli.net/2023/11/10/m9KjLE6YukPpdsz.png)
可以看到该对象中还保存了一个对象指针`parent`，分支下的每一次commit都有先后时间，根据时间顺序，保存每次修改的对象就形成了一条时间链。`parent`保存了时间链中，当前对象的上一个对象的指针
通过`parent`指针，我们就能不断地往前追溯历史commit

我们还可以在master主分支上创建分支，并对该分支提交修改，此时也将形成一条版本链

执行指令，查看当前git仓库中存在哪些分支
```bash
git branch -a
```
`-a`将显示本地分支与远程分支，默认只显示本地分支
![image.png](https://s2.loli.net/2023/11/10/xMFnf1gyVDPolmI.png)

除了master分支，`HEAD`也能指向其他分支，由`HEAD`指向的分支被称为工作分支，也就是当前分支
执行命令，创建新的分支：
```bash
git brach dev
```
![image.png](https://s2.loli.net/2023/11/10/Y8taRmiG96Euw1z.png)

创建新的分支后的目录结构：
![image.png](https://s2.loli.net/2023/11/10/7YIWnfk1SLCoHrB.png)
此时.git目录下出现了dev文件，查看该文件中的内容
![image.png](https://s2.loli.net/2023/11/10/UvYH7Llzsahg9WK.png)
发现它和master文件的内容一样，由此证明新建分支的数据基于原分支

切换分支
```bash
git checkout 分支名
```
![image.png](https://s2.loli.net/2023/11/10/e1ukDvfzctnMAxw.png)
\* 在dev之前，说明当前`HEAD`指针指向了dev分支

也能使用一行命令完成分支的创建与切换：
```bash
git ckeckout -b 分支名
```
在dev分支中，修改ReadMe文件，并切换到master分支，查看ReadMe文件
![image.png](https://s2.loli.net/2023/11/10/Mbw6W9XRHUcdNIa.png)
发现只有在dev分支中才能看见修改后的ReadMe文件，而在master分支中，无法看见修改后的ReadMe文件
说明此次的修改被提交到了dev分支上，对master分支不可见

同理，在dev分支上提交数据后，refs/heads/dev保存的对象指针和refs/heads/master保存的对象指针不再相同
![image.png](https://s2.loli.net/2023/11/10/vCWuzB9f4ZsJaFI.png)

如何合并分支？首先要切回master分支
```bash
git checkout master
git merge 分支名
```
![image.png](https://s2.loli.net/2023/11/10/bL5jF43vR1IhdrE.png)
合并分支后，打印信息说，master分支下的ReadMe文件新增了一行数据
### 分支的删除
不能删除当前所在分支（工作分支），只能删除其他的分支
```bash
git branch -d 分支名
```
合并分支与删除分支的速度非常快。因此，git鼓励我们在分支上完成任务，合并到master分支中再删除分支，这样将更加安全

如果分支执行了add或者commit，但是没有被合并，此时无法使用`-d`删除该分支（git认为创建出来的分支都是要被merge的，只有在merge分支后，才能删除该分支），只能使用`-D`强制删除该分支
```bash
git branch -D 分支名
```
### 合并分支时的冲突
在dev分支中修改ReadMe文件，也在master分支中修改ReadMe文件，使两份文件不同
修改完文件我们要进行add和commit，使仓库中两个分支下的文件内容不同
![image.png](https://s2.loli.net/2023/11/10/jBSJgDMLycvFT2n.png)

此时执行合并操作
![image.png](https://s2.loli.net/2023/11/10/VHPWc2NrtjUOlBD.png)
打印信息说合并失败，此时查看ReadMe文件
其中`<<<<<<HEAD`到`=======`之间为当前分支下的数据，`========`到dev之间为dev分支的数据，两者不相同所以产生了冲突

如何解决冲突？手动解决，在master分支中的冲突文件中保留想要的代码。解决冲突后执行add和commit，此时**合并操作**才算完成
![image.png](https://s2.loli.net/2023/11/10/1QO2ASXMurLyNkK.png)

查看当前日志
![image.png](https://s2.loli.net/2023/11/10/yYKzgle1cIfJPk8.png)

使用以下命令，可以以图表的形式看到master分支与dev分支之间的关系
```bash
git log --graph --abbrev-commit
```
![image.png](https://s2.loli.net/2023/11/10/F5HRdUscegLVaOA.png)
#### 分支的合并模式
如果执行命令
```bash
git merge 分支名
```
调试信息中有"fast-forward"字段时，说明这次的提交为ff模式
该模式的合并无法通过`git log --graph --abbrev-commit`查看，图中不会出现两个分支合并成一个分支的信息，因此我们无法得知master合并了哪个分支

ff模式的本质是将被合并分支的**最近一次提交对象指针**写入到master分支的**最近一次提交对象指针**中
![image.png](https://s2.loli.net/2023/11/10/bL5jF43vR1IhdrE.png)

当合并没有发生冲突时，git默认使用ff模式
此外，还有一种模式： no-ff，这种模式的合并可以通过`git log --graph --abbrev-commit`知道两哪个分支发生了合并
使用no-ff模式的合并：
```bash
git merge -no-ff -m "此次合并的目的" 被合并的分支名
```
由于合并完分支后，还需要进行commit操作，所以需要`-m`提交信息
#### 分支策略
master主分支对应着线上环境，必须是十分稳定的，所以提交的代码需要能稳定运行，没有重大的bug
而其他分支则是不稳定，存在bug的。开发人员在这些分支上进行开发，进行一系列测试验证后，就能将其他分支上的代码合并到master分支中
基于分支，我们就能进行同一项目的多人同时开发，同时也演变出了多种分支策略
#### git stash
如果在其他分支中修改了文件，未进行add和commit，此时的操作将不受到git的版本控制，只是单纯的对工作区的修改。切回master分支后，可以在本地看见其他分支对文件的修改

若其他分支修改文件后，进行了add和commit，此时的操作将受到git的版本控制。切回master分支后，无法看见其他分支对文件的修改

如果不进行add和commit，又不想让master分支看见其他分支对文件的修改，可以使用命令
```bash
git stash
```
该命令将保存分支对文件的修改到一个特殊文件中，也就是将修改后的文件添加到git的版本控制中，此时master分支无法看见其他分支对于文件的修改
![image.png](https://s2.loli.net/2023/11/10/sfeUJ3NBxKPcIpT.png)

此时.git的目录树中，多了一个stash文件，用来保存分支中的修改
![image.png](https://s2.loli.net/2023/11/10/NLHFj3OkftmguZ4.png)

执行命令，打印stash中保存的修改
```bash
git stash list
```

如果在dev分支中修改了文件，并执行`git stash`，切出该分支，再切回dev分支
此时分支中的数据和master分支相同，要想恢复之前所做的修改，就需要执行命令，将stash中的修改恢复到分支中
```bash
git stash pop
```
![image.png](https://s2.loli.net/2023/11/10/IVDqn7Sr2WMZEJ8.png)
#### 不要在master分支上解决冲突
项目上线后可能遇到的场景：存在一个小bug，此时基于master分支创建分支`fix_bug`进行bug修复，修复成功后在master分支中合并fix_bug分支，项目再次上线
同时，项目的开发也在不断进行中，在发现bug之前，创建了一个分支dev开发新的功能。等到功能开发好时，项目已经不再是当时创建分支时的项目，而是修改了bug后的项目。此时在master分支中合并dev分支将产生冲突
![image.png](https://s2.loli.net/2023/11/10/C6FXV349aL8swti.png)
我们不能直接合并dev分支，并在master分支上手动修改文件，这可能导致使修改后的文件出现bug
正确做法应该是：在dev分支上合并master分支并且手动修改冲突，测试没有问题后，再到master分支中合并dev分支

## git的远程操作
git的全程叫做分布式版本控制系统，版本控制很好理解，如何理解分布式？

一个项目的开发往往需要多人的分工协作，之前所说的git仓库：工作区、暂存区以及版本库都是在本地运行的。一个开发人员对项目进行了修改，另一个开发人员要如何获取到他的修改呢？或者说如何让其他人看到我对项目的修改？

显然只使用本地的git无法解决这个问题。git为我们提供了一个解决方案，使用中央服务器存储git仓库，我们也叫它远程仓库

每个开发人员都能访问到远程仓库，需要开发项目时，从远端仓库将整个仓库拉取下来进行开发。开发完成后，将本地的修改提交到远程仓库中。那么其他开发人员就能通过远程仓库看到你对项目的开发，并且也能拉取你对项目的修改，进一步进行开发

用https克隆远程仓库，注意：不能在拥有.git目录的路径下（已经初始化过本地仓库的地方）克隆仓库！
克隆也能理解为第一次拉取

在远程仓库中可以找到该仓库的链接，通常有https和ssh协议，先演示https协议的克隆，将URL复制下来，执行命令
```bash
git clone URL
```
![image.png](https://s2.loli.net/2023/11/10/hJVwgIdPTtBWumF.png)

查看远程仓库的信息
```bash
git remote [-v]
```
直接使用`git remote`将打印本地仓库中已经配置的远程仓库名字，通常是origin
![image.png](https://s2.loli.net/2023/11/10/z9VFqiNps8Mn25X.png)
而`git remote -v`将查看已经配置的远程仓库列表，以及它们的URL。`fetch`表示本地拥有远程仓库的拉取权限，`push`则表示提交权限

用ssh克隆远程仓库：
需要先将本地服务器的公钥放到远端的git服务器中

如何配置git服务器的公钥？
找到个人设置中的相关配置项
![image.png](https://s2.loli.net/2023/11/10/Td2VFZOIwbhze7u.png)
点击添加公钥：
![image.png](https://s2.loli.net/2023/11/10/dQq9rK3OwbpmJFZ.png)

在本地服务器中生成公钥与私钥：
进入/~/.ssh目录，如果没有则创建目录
如果目录中没有id_rsa 和id_rsa.pub文件，则执行以下命令
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
其中`-C`选项表示添加注释信息，用来表示生成的密钥对的用途，将这个信息设置为自己的邮箱即可
输入命令后一直回车直到能够输入其他命令为止
![image.png](https://s2.loli.net/2023/11/10/J9uUXLBKOoNVpEg.png)

此时目录下新增两个文件：id_rsa 和id_rsa.pub
![image.png](https://s2.loli.net/2023/11/10/1Et59DmUcx63qeZ.png)
生成后，将公钥（id_rsa.pub中的字符）写入git服务器中
![image.png](https://s2.loli.net/2023/11/10/H37MWAYCxQqNEdf.png)

此时执行命令，即可克隆仓库
```bash
git clone ssh链接
```
![image.png](https://s2.loli.net/2023/11/10/rjnVTY54u8JpvDB.png)

### 将修改提交到远程仓库
注意：不是将整个本地仓库的修改提交到远程仓库，而是将某个分支的修改提交到远程仓库
克隆完远程仓库后，需要配置user.name和user.email，这两个信息要和git服务器中的信息一致

首先在工作区中修改文件，并add与commit，最后执行push，将版本库中的修改提交到远程仓库

```bash
git push 仓库名 本地仓库的分支:远程仓库的分支
```
push其实做了两步操作：将本地仓库某个分支的修改提交到远程仓库中，将指定分支与远端分支进行合并

其中，仓库名是本地服务器配置的远程仓库的名字，一般是`origin`
同时指定要push哪个本地分支中的修改到远程仓库中的哪个分支，这两个分支名一般用`:`隔开，当两者相同时，可以只写一次分支名（github的master分支现在叫做main分支）
![image.png](https://s2.loli.net/2023/11/10/8wCBc5HgZipSIf2.png)

此时远程仓库中出现了分支的修改，新增了ReadMe文件
![image.png](https://s2.loli.net/2023/11/10/mNLMVduycsqEjrX.png)

push操作的本质是本地分支与远端分支的交互，交互之前需要先建立连接。一般在克隆仓库时，本地仓库的master分支与远程仓库的master分支就自动建立了连接，可以直接使用`git push`。而其他分支没有建立连接，需要使用上面的长命令进行`push`操作

当本地仓库的版本领先于远程仓库的版本，此时进行push操作以推送更新
当远程仓库的版本领先于本地仓库的版本，此时要进行pull操作以同步与远程仓库的版本

假设远程仓库的hello.txt被其他人修改了并且push了，这是修改后的内容
![image.png](https://s2.loli.net/2023/11/10/3zCc6ybK9wkiars.png)

此时我要进行下一步的开发，就必须与远程仓库的版本保持同步
使用pull命令完成同步
```bash
git pull 仓库名 本地仓库的分支:远程仓库的分支
```

pull和push的使用相同，pull其实也进行了两步操作：将远程仓库中指定分支的修改拉取到本地，与本地分支进行合并
![image.png](https://s2.loli.net/2023/11/10/o1OGhDueTiy7nv2.png)
### push时忽略特殊文件
创建远程仓库中，可以勾选添加.gitignore文件的选项：指定一些不想被push的文件，这些文件的修改就不会被add到暂存区中，更不会被push到远程仓库中了
在.gitignore文件中的文件不会被git追踪管理，即忽略这些文件，.gitignore文件需要位于git工作区的根目录下

若新建仓库时没有勾选添加.gitignore文件的选项，后续想要添加该文件时，可以直接添加
![image.png](https://s2.loli.net/2023/11/10/L439f6qkvDXpirT.png)
该文件在工作区的根目录下
![image.png](https://s2.loli.net/2023/11/10/QN5eWuKVqEiDHLn.png)

此时创建一个以so结尾的文件，并将当前目录下的所有文件add到工作区中，执行`git status`命令，查看哪些文件的修改需要被commit到版本库中
![image.png](https://s2.loli.net/2023/11/10/VC8nSL3lr9gXwZx.png)

发现调试信息中，没有`a.so`文件只有`.gitignore`文件，说明`.gitignore`文件生效了

如何强制添加文件到暂存区中，就算该文件在.gitignore中出现？
为add添加`-f`选项，强制提交修改到暂存区中
```bash
git add -f filename
```
![image.png](https://s2.loli.net/2023/11/10/kq7WEAbPY3jV4Oe.png)
也可以在`.gitignore`文件中添加规则，使git不要不略某个文件的提交
![image.png](https://s2.loli.net/2023/11/10/gkSyetPHuR94nYC.png)
`!`表示不忽略该文件的提交

如何判定某一文件是否被忽略？
```bash
git check-ignore -v filename
```
![image.png](https://s2.loli.net/2023/11/10/hfG3VYr7wzSeTJs.png)
打印信息说，该文件被第三行的规则忽略了
#### 配置命令的别名
```bash
git config --global alias.别名 '原名（不需要加git）'
```
给status起别名
![image.png](https://s2.loli.net/2023/11/10/vWCUYxjqneyriDo.png)
## git的标签管理
使用标签可以方便的查看信息，以及记录一些有意义的信息
执行命令，为最近的一次commit将被打上指定的标签
```bash
git tag 标签名
```

查看所有标签
```bash
git tag
```
![image.png](https://s2.loli.net/2023/11/10/v8ko9IcK3VE7uQZ.png)

.git的目录树中也出现了相应的目录，查看该文件
![image.png](https://s2.loli.net/2023/11/10/Ye1FTu3BANXQWUD.png)
发现该文件存储了最近一次的commit id

为指定的commit打标签，先通过`git log --pretty=oneline --abbrev-commit`，列出所有commit id，找到想要打标签的commit id
![image.png](https://s2.loli.net/2023/11/10/povfQzkyhsIqcYC.png)

执行
```bash
git tag 标签名 commitid 
```

执行命令，为commit打标签并添加注释：
```bash
git tag -a 标签名 -m "注释内容" commitid
```

执行命令，查看标签的注释：
```bash
git show 标签名
```
`git show`将显示对应commit的详细信息，如果该commit的标签有注释，那么也将一起展示，如果没有则只显示其他信息
![image.png](https://s2.loli.net/2023/11/10/agSvzLK38R2Bhni.png)

删除标签：
```bash
git tag -d 标签名
```
### 推送标签
```bash
git push 仓库名 标签名
```

推送所有标签：
```bash
git push 仓库名 --tags
```
![image.png](https://s2.loli.net/2023/11/10/xkZWTUvjNArzlYF.png)

删除远端仓库的标签（首先要先删除本地仓库的标签）
```bash
git push 仓库名 :标签名
```
## git实战
### 同一分支下的多人协作（不推荐）
开发者1在dev分支下的`hello.txt`文件中添加"111"，开发者2在dev分支下的test文件中添加"222"

先在远端仓库中添加新的分支
![image.png](https://s2.loli.net/2023/11/10/qOxFKuYTAEMQHoe.png)

接着在两个开发者的机器上执行`git pull`，获取最新的仓库版本（新的分支）
但是执行`git branch`后，无法看到新的分支。这是因为远端的分支与本地的分支没有关系，所以就算拉取了，也不会执行分支合并的操作。使用`git branch -r`选项就能看到远端的分支
![image.png](https://s2.loli.net/2023/11/10/Z6YdeJBIhRX4ToH.png)
此时我们要在本地手动创建名字相同的分支，并将本地分支与远端分支**建立连接**
执行命令建立连接
```bash
git checkout -b 本地分支名 仓库名/远端分支名
```
除此之外，使用其他命令也能达到同样效果（前提是dev分支不存在）
```bash
git branch 本地分支名 仓库名/远端分支名v
git checkout 本地分支名
```
若dev分支存在，则执行
```bash
git branch --set-upstream-to=仓库名/远端分支名 本地分支名
git checkout 本地分支名
```
![image.png](https://s2.loli.net/2023/11/10/AkOnJhc7LumqxQY.png)
执行`git branch -vv`可以查看本地仓库与远端仓库的连接
为什么要建立连接？这样可以直接使用`git push`与`git pull`这样的短命令，而不使用`git push origin main`这样的长命令

开发者1完成操作，修改文件后add、commit、push三板斧
![image.png](https://s2.loli.net/2023/11/10/8CvcXDZfi4wLSsM.png)

开发者2执行相同的操作
![image.png](https://s2.loli.net/2023/11/10/uMCk7UvP5cY4ZTn.png)
而开发者2修改完hello.txt却无法提交，原因是此时的远端仓库中hello.txt和开发者2本地仓库最近一次pull的版本不同（开发者1的提交导致版本的不同），此时提交将产生冲突
解决方法是：执行`git pull`，此时`hello.txt`文件产生冲突，手动修改冲突并重新add、commit、push即可

![image.png](https://s2.loli.net/2023/11/10/N1QsH8Atirq6f2p.png)

最后一步：将dev分支合并到master分支中
有两种方法：
1. 将本地的dev分支合并到master分支中，再将master分支push到远端
2. 在git服务器上提交pr（pull request）申请，由管理员审批后，合并将被执行

如何提交pr申请？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311131019257.png)
选择选项，将dev分支合并到main分支中，添加标题和内容后选择reviewers和addignees，点击新建pr
关于reviewers和addignees的区别，可以看这篇博客[Git GitHub上的Reviewers和Assignees之间有什么区别|极客笔记 (deepinout.com)](https://deepinout.com/git/git-questions/288_git_what_is_difference_between_reviewers_and_assignees_on_github.html)![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311131026659.png)
相关人员审查没问题后，就会进行合并操作了

另一种方法是在本地执行merge操作，推荐使用pr合并方式，经过审查员的审查，合并操作会更加安全

首先，执行合并操作的开发人员的本地仓库中，dev分支可能不是最新，此时需要先pull拉取最新版本
```bash
git checkout dev
git pull
```
由于无法保证main分支也是最新的，所以也需要pull
```bash
git checkout main
git pull
```
由于无法保证将dev分支合并到main分支是否会发生冲突，所以先到dev分支下，将main分支合并到dev分支。若发生冲突则手动解决冲突
```bash
git checkout dev
git merge main
```
解决完冲突后，再回到main分支下，合并dev分支
```bash
git checkout main
git merge dev
```
最后进行push操作即可
```bash
git push
```
删除dev分支（可选）
```bash
git branch -d dev
```

可以看到同一分支下的多人协作非常麻烦，产生冲突的概率极大，需要不停的pull解决冲突
所以开发中的多人协作一般是在不同分支下进行的
### 不同分支下的多人协作
每个开发者（开发项目的不同功能）对应不同分支，在属于自己的分支上提交自己开发的代码
最后这些代码将合并到main分支下
如：开发者1在feature-1分支下提交file1文件，开发者2在feature-2分支下提交file2文件
开发者1：首先pull更新本地的master分支，然后在本地master分支下，创建feature-1分支，编辑文件file1并进行add和commit操作
```bash
git pull
git checkout -b feature-1
```
此时的push操作比较特殊，不能直接`git push`，而要执行
```bash
git push 远端仓库名 本地分支名
```
因为此时的远端仓库中没有feature-1分支，我们只是在本地创建了feature-1分支，所以这次的提交需要将整个分支进行提交
总结下：可以在本地创建新分支，也可以在远端创建新分支
1. 在本地创建新分支后，需要将整个分支进行提交
2. 而在远端创建新分支后，需要先pull获取远端的分支，再创建本地的同名分支并建立连接
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311131104419.png)
此时远程仓库下已经存在feature-1分支了
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311131105861.png)

开发者2执行与开发者1相同的操作：基于master分支创建新的分支feature-2，在该分支上编写文件file2，并执行add、commit以及push操作，在执行push操作之前需要先pull获取最新的远程仓库

开发过程中可能遇到的场景：开发者1需要帮助开发者2完成后续开发
此时开发者1的本地仓库中没有featrue-2分支，所以需要将该分支拉取下来
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311140837809.png)
解释一下这时的pull为什么只需要使用短链接？
pull操作其实有两种：1. 拉取某一分支下的内容，此时需要使用长连接，如果本地分支与远端分支建立连接后即可使用短连接 2. 拉取仓库中的内容，如新的分支，此时使用短连接即可，但是拉取到的新分支不会和本地分支建立连接

但是拉取操作只是将远端的分支拉取下来，本地没有相应的分支，打印的信息提示我们要在本地创建相同的分支并与其链接
 ```bash
 git checkout -b feature-2 origin/feature-2
```
此时处于feature-2分支下，继续编写该分支下的文件，完成开发者2未完成的代码

最后：将feature-1分支和feature-2分支合并到main分支下
在git服务端分别提交两次pr申请即可
在提交pr申请前，推荐先用其他分支合并main分支，解决可能存在的冲突，再用main分支合并其他分支，此时将不会产生冲突

### 为什么本地依然能看到远端已经删除的分支？
将远端分支删除后，本地使用`git branch -a`依然能看到远端分支，如何解决？
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311140907185.png)
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311140908997.png)

使用命令清理远端无用分支即可
```bash
git remote prune origin
```
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311140909147.png)
此时远端分支只有main分支，剩下的本地分支用`git branch -d`删除即可
### git flow模型
git flow模型是企业级常用的一种git分支设置规范，这只是一种常用的开发模型，并不适合于所有的开发团队。了解它有助于我们理解软件开发的流程
![image.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/202311140945542.png)

|分支|名称|适用环境|
|-|-|-|
|master|主分支|生产环境|
|release|预发布分支|预发布/测试环境|
|develop|开发分支|开发环境|
|feature|需求开发分支|本地|
|hotfix|紧急修复分支|本地|
**master分支**
- master为主分支，该分支唯一且只读，用于部署正式发布环境，一般由合并release分支得到
- master分支为稳定的代码库，任何时候都不能直接在master分支上修改代码
- 产品的功能全部实现后，最终在master分支上对外发布。并且master分支上的所有提交都应该打标签（tag）做记录，方便追溯
- master分支不可删除

**feature分支**
- feature为新功能或新特性的开发分支，以develop分支为基础创建feature分支
- 命名通常为`feature/user_createtime_feature`，表示开发人员，开发时间以及所开发的特性
- 新特性开发完成后需要将feature分支合并到develop分支中，且删除feature分支

**develop分支**
- develop为开发分支，基于master分支创建的唯一且只读分支，始终保持最新完成以及bug修复后的代码。可部署到对应开发集群
- 通常用来合并feature分支，也可直接在develop分支上开发

**release分支**
- release分支为预发布分支，基于develop分支创建，可以部署到测试或者预发布集群
- 通常的命名规则：`release/version_publishtime`，表示发布的版本以及发布时间
- release分支通常用于提交给测试人员进行测试，若出现问题则需要验证develop是否出现问题，测试完成后可以删除release分支

**hotfix分支**
- hotfix为bug修复分支或补丁分支，主要用于线上版本的bug修复
- 命名规则通常为`hotfix/user_createtime_hotfix`，表示相关人员，创建时间以及具体的bug
- 问题修复完成，将其合并到master分支后删除


