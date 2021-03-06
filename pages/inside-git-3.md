# 认识Git对象

---

> 探索Git背后的秘密，认识Git引用，熟悉各种引用类型，理解HEAD文件的作用

> 也许你早已经熟悉了Git的日常使用，但是你可曾想过：为什么每次新建Git库时都要执行`git init`呢？执行`git init`后生成的.git目录里到底藏了哪些秘密？平常使用Git客户端，以及命令行执行git命令时，Git在背后到底为我们默默地做了些什么呢？阅读本文以后，一切谜团都将引刃而解！

注：
本文的大部分写作灵感来自于[“Pro Git book”](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)。感谢原作者的精彩分享。
本文采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a>

![](https://morningspace.github.io/assets/images/lab/git/logo-3.png)

## 认识Git引用

前面我们介绍了Git对象的基本概念，以及几类典型的Git对象。从本节开始，我们将把目光投向Git引用（Reference），也就是指向Git对象的指针。

我们已经看到，在`git log`后面加上某个commit对象的唯一键，可以查看从这个提交开始往前的所有提交历史。但是，这依然需要我们记住commit对象的唯一键才行。如果能把唯一键存到某个文件里，然后利用这个文件的文件名来引用相应的commit对象，就不需要我们记住那些毫无规律的hash值了。事实上，Git就是这么做的。在Git里，这些引用被保存在.git/refs目录下。
```shell
$ find .git/refs
.git/refs
.git/refs/heads
.git/refs/tags
$ find .git/refs -type f
```

观察.git/refs目录我们可以发现，refs下面还有两个子目录，分别是heads和tags。关于heads和tags，后面我们还会详细讨论。由于目前为止，我们还没有创建任何引用，所以.git/refs下是一套空的目录结构。下面我们就来手工新建一个引用，让它指向当前分支（默认即master分支）的最新提交：
```shell
$ echo e0ec828eda6b51b170fff6b5fdfa03a3cb70a13e > .git/refs/heads/master
```

这样我们就可以用`git log`来查看master分支上从head指针开始往前的所有提交历史了：
```shell
$ git log --pretty=oneline master
e0ec828eda6b51b170fff6b5fdfa03a3cb70a13e (HEAD -> master) third commit
04081ddb43269238a1cb8a61a2d04a36986febfa second commit
4c50701f89265f9ca6eeb3ddffae450da55f9bd5 first commit
```

另外，Git还专门提供了一个`git update-ref`命令，也可以用它来给引用设值：
```shell
$ git update-ref refs/heads/master e0ec828eda6b51b170fff6b5fdfa03a3cb70a13e
```

实际上，Git的分支功能就是基于引用实现的。Git在每个分支上都有一个head指针，指向该分支的最新提交。比如，如果我们要在第二次提交的地方新开一个分支，可以这样做：
```shell
$ git update-ref refs/heads/dev 04081ddb43269238a1cb8a61a2d04a36986febfa
```

结果相当于我们用`git branch`命令新建了一个名为dev的分支。如果这个时候执行`git branch`，我们会看到除了master分支外，还多了一个dev分支：
```shell
$ git branch
* master
  dev
```

用`git log`查看dev分支的提交历史，我们会发现这次的提交历史是从第二次提交开始的，这是因为第三次提交只存在于master分支上：
```shell
$ git log --pretty=oneline dev
04081ddb43269238a1cb8a61a2d04a36986febfa (dev) second commit
4c50701f89265f9ca6eeb3ddffae450da55f9bd5 first commit
```

Git引用与相应commit对象之间的关联关系如下图所示：

![](https://morningspace.github.io/assets/images/lab/git/inside-git-3.png)

## HEAD指针

前面提到了，利用Git引用可以帮助我们创建分支。所以，位于.git/refs/heads下的引用又被称为branch引用。有一个问题，当我们执行`git branch <branch>`新建分支的时候，Git是怎么知道最新提交所对应的唯一键的呢？答案就在.git/HEAD文件里。按照Git的说法，HEAD文件是一种指向当前分支上最新提交的符号引用（symbolic reference）。所谓符号引用是指，它并不直接指向某个commit对象，而是指向另一个引用，再由那个引用间接地指向commit对象。比如，如果我们查看.git/HEAD文件的当前内容：
```shell
$ cat .git/HEAD
ref: refs/heads/master
```

会发现它的值等于refs/heads/master，代表了master分支上的head指针。再通过查看.git/refs/heads目录下的master文件，我们就能找到对应的commit对象了：
```shell
$ cat .git/refs/heads/master 
e0ec828eda6b51b170fff6b5fdfa03a3cb70a13e
```

如果这个时候我们把分支从master切换到dev，然后再看HEAD文件的内容：
```shell
$ git checkout dev
Switched to branch 'dev'
$ cat .git/HEAD
ref: refs/heads/dev
```

就会发现HEAD文件所对应的引用已经变成了refs/heads/dev，也就是dev分支上的head指针。HEAD文件与相应Git引用之间的关系如下图所示：

![](https://morningspace.github.io/assets/images/lab/git/inside-git-4.png)

另外，当我们执行`git commit`时，Git在创建commit对象时也会更新HEAD文件，让head指针指向新的commit对象，并把HEAD文件里原来所指向的commit对象作为新commit对象的parent。

对于符号引用的操作，Git还提供了一个专门的命令：`git symbolic-ref`。利用它，我们不仅可以实现对HEAD文件内容的读取，还可以进行安全的写入：
```shell
$ git symbolic-ref HEAD
refs/heads/master
$ git symbolic-ref HEAD refs/heads/dev
$ cat .git/HEAD
ref: refs/heads/dev
```

## Tag对象及引用

Tag是Git提供的又一种对象类型，它和commit对象有点类似，包含了：tag添加者的信息，tag的添加日期，相关注解信息，以及一个指向commit对象的引用。和commit对象不同的是：tag对象通常指向的是一个commit对象，而commit对象则指向tree对象。Tag有点类似于commit对象的别名，并且它和commit对象之间的这种对应关系一旦定义，就不会再变了。

Git支持两种类型的tag：annotated和lightweight。我们可以利用`git update-ref`很轻松地创建出一个lightweight类型的tag：
```shell
$ git update-ref refs/tags/v1.0 04081ddb43269238a1cb8a61a2d04a36986febfa
```

实际上，这种tag只是一个指向相应commit对象的引用。查看.git/refs/tags目录下文件v1.0的内容会发现，里面所包含的引用指向的是我们之前的第二次提交：
```shell
$ cat .git/refs/tags/v1.0
04081ddb43269238a1cb8a61a2d04a36986febfa
$ git cat-file -p 04081ddb43269238a1cb8a61a2d04a36986febfa
tree 955f6fef4f43ee1f5d93cbea718cce3048450f4b
parent 4c50701f89265f9ca6eeb3ddffae450da55f9bd5
author dev <dev@example.com> 1556719543 +0000
committer dev <dev@example.com> 1556719543 +0000

second commit
```

创建annotated类型的tag，则要稍微复杂一些。Git首先会建立一个tag对象指向某个commit对象，然后在.git/refs/tags目录下的对应文件里写入指向该tag对象的引用，而不是直接指向commit对象。下面我们就利用`git tag -a`来创建一个annotated类型的tag：
```shell
$ git tag -a v1.1 07ef9d54dd0da246d069dfa2ad2350751203ecb2 -m 'include bak'
```

来看一下.git/refs/tags目录下文件v1.1的内容：
```shell
$ cat .git/refs/tags/v1.1
6d9cfead57862bc571d52d11f54f25789a103513
```

这是一个指向tag对象的引用：
```shell
$ git cat-file -p 6d9cfead57862bc571d52d11f54f25789a103513
object 07ef9d54dd0da246d069dfa2ad2350751203ecb2
type tree
tag v1.1
tagger dev <dev@example.com> 1556746261 +0000

include bak
```

这个tag对象指向的是另一个commit对象，代表了当前分支的最新提交，即：“third commit”所对应的commit对象。

## Remote引用

Remote是一种特殊类型的引用，对应于远程Git库的提交。如果我们添加了一个remote类型的引用，并且把本地变更通过`git push`推送到了远程，Git就会在.git/refs/remotes目录下新建一个文件，针对当前分支最近一次推送所对应的commit对象，存入该对象的唯一键。

和前面提到的branch引用相比，remote引用是相对“只读”的。对于branch引用而言，当我们执行`git checkout`切换分支后，引用的值就变了。而对于remote引用而言，因为它们所指向的是远程Git库对应分支下的最新提交。只要这个信息在远程库里没有改变，那么本地的引用也是不会变的。

下面，我们来添加一个名为origin的remote引用，并把我们在本地master分支上的改动推送到远程。

如果我们使用的是[Hello Git](https://github.com/morningspace/lab-hello-git)提供的Docker镜像，那么首先需要在代表远程Git服务的容器里创建好与本地同名的远程库。为此，我们需要在代表本地Git客户端的容器里通过SSH以用户git的身份登录到远程Git服务，并利用Git Shell命令创建远程库：
```shell
$ ssh git@my-git-remote
git> create inside-git
Initialized empty Git repository in /home/git/inside-git.git/
git> list
inside-git.git
hello-git.git
git> exit
```

退出远程Git服务，并回到本地Git客户端后，我们就可以把本地的当前变更推送到远程库了：
```shell
$ git remote add origin git@my-git-remote:~/inside-git.git
$ git push origin master
Counting objects: 9, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (5/5), done.
Writing objects: 100% (9/9), 664 bytes | 221.00 KiB/s, done.
Total 9 (delta 1), reused 0 (delta 0)
To my-git-remote:~/inside-git.git
 * [new branch]      master -> master
```

这个时候，Git就会在.git/refs/remotes/origin目录下新建一个名为master的文件。查看其内容会发现，它所指向的正是我们在master分支上的最新提交，也是我们向远程Git服务推送更新时所对应的提交：
```shell
$ cat .git/refs/remotes/origin/master
e0ec828eda6b51b170fff6b5fdfa03a3cb70a13e
```
