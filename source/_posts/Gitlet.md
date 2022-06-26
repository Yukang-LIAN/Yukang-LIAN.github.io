---
title: Gitlet
date: 2022-06-25 10:18:53
tags:
---
## 前言

最近在学习UCB的CS61b课程，我学习的版本是2018Spring，用了大约一个月完成了所有Lab、Homework和Project，之前就听群友说61b的gitlet项目非常值得做，但是2018Spring版本没有，于是找来了21Spring的gitlet写了一下，前前后后写了6天总共花了37个小时（仅编代码与debug时间），代码大约1200行，其中有大概10小时花在给`merge`函数debug，只是我当时脑子不清醒逻辑有点混乱所以才导致花了这么长时间，其实只是几行代码顺序写反了，找bug找了很久:sob:。附上时间、分数以及[Github链接](https://github.com/Yukang-LIAN/005-UCB-CS61B-GITLET/tree/main/proj2)。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206242136798.png"  style="zoom: 40%;" />
    </br>
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206242135232.png"  style="zoom: 40%;" />
</div>

## Gitlet总体介绍

Gitlet是UCB CS61b课程中的一个项目，实现了简化版的git，支持`add`，`commit`，`log`，`checkout`，`merge`等操作。git是一个分布式的版本控制工具，有一定编程经验的小伙伴肯定对这个软件比较熟悉，我只会`add`，`commit`，`push`，`pull`，这对我来说就已经够用了:satisfied:，如果想要了解更多的git知识，请点击[git官网](https://git-scm.com)查看。

几个可能有帮助的资料：[gitlet官方文档](https://sp21.datastructur.es/materials/proj/proj2/proj2)，[gitlet学习笔记](https://blog.csdn.net/weixin_43405649/article/details/124270510)，[知乎git动图详解](https://zhuanlan.zhihu.com/p/96631135)。对了，在做gitlet请前先把[Lab6](https://sp21.datastructur.es/materials/lab/lab6/lab6)完成，熟悉文件的读写操作。

git和gitlet在原理上差不多，但实现上gitlet简化了不少内容，时间不充裕的话不建议钻牛角尖实现真正的git，只需要通过这个项目了解一下git大致原理就好。

## Gitlet原理

原理上gitlet与git并无二致，都是将一个版本的所有文件内容保存起来，可以在不同版本之间进行切换，保证所有存在过的版本都不会丢失，那如何区分一个版本呢？答案是用commit命令创建一个版本快照（snapshot），就相当于用相机拍下这个时刻的照片，并且按顺序将照片串起来，当想回到某个版本时，只需要按照顺序往前翻就可以了。下图展示了Gitlet的简单原理。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/GitletOverviewFig1.png"  style="zoom: 60%;" />
</div>

当Gitlet初始化时，系统自动生成一个init Commit，这个Commit不记录任何文件，之后，每有一个新的版本，会有一个新的Commit，Commit会记录版本文件信息，也就是图中的指针指向的两个文件。

当前最新版本的Commit会由一个Head进行标记，此处是Commit2，也就是说当前文件夹内只包含`3.txt`和`4.txt`，当需要切换到Commit1时，我们可以看到Commit1记录了`1.txt`和`2.txt`，那如何进行版本转换呢？只需要将Head移动到Commit1上，读取Commit1记录的文件`1.txt`和`2.txt`，将Commit1记录的文件重新写入到当前文件夹，并且删除`3.txt`和`4.txt`即可完成版本转换，现在的文件夹就只包含`1.txt`和`2.txt`了。

上图中Commit1和Commit2内容完全不同，如果Commit1和Commit2存在相同的文件，只需要保存一个就可以，这样能够在很大程度上节省空间。如下图所示。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/GitletOverviewFig2.png"  style="zoom: 60%;" />
</div>

这样转换版本时只需要将`4.txt`删除，添加新的`1.txt`就可以了。

gitlet还可以实现不同的分支功能，当多人进行同时开发时，为了互不影响，可以采取多个路线同时进行的方式，在gitlet中叫做`branch`，如下图所示，BranchA和BranchB是两个不同的分支，HeadA和HeadB是为了记录当前gitlet的所有分支的头，而Head则指向当前的Commit，彼此之间没有关联。也就是说，当前文件夹中的文件依然只有`2.txt`和`4.txt`，HeadB和HeadA只是起到一个记录的作用，并不代表文件夹当前的Commit。例如，Head和HeadA可以同时指向Commit3A。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206242116786.png"  style="zoom: 50%;" />
</div>

## Gitlet内部结构实现

接下来说一下gitlet的内部实现，首先，`gitlet init`命令会创建`.gitlet`文件夹，`.gitlet`文件夹是一个隐藏文件，windows要先打开显示隐藏文件的设置，linux或macox可通过`ls -la`进行查看，gitlet绝大部分操作都在`.gitlet`文件夹中进行工作，此文件夹的结构如以下代码所示。

``` java
/*
     *   .gitlet
     *      |--objects
     *      |     |--commit and blob
     *      |--refs
     *      |    |--heads
     *      |         |--master
     *      |--HEAD
     *      |--addstage
     *      |--removestage
     */
```

### objects
`objects`是gitlet中一个相当重要的概念，每一个Commit对应一个Commit对象，每一个文件对应一个Blob对象，如何定义一个对象呢，答案是由它本身的`hashcode`来决定的，也就是根据每个对象不同的信息，通过`sha1`算法来得到独一无二的一个`hashcode`，我把这个`hashcode`叫做ID，只有拥有同一个ID的才能视为同一对象，这些对象都通过序列化写入到objects文件夹中，每个对象对应一个文件，文件名即为40位的对象ID。

对于每个文件来讲，在`gitlet`中是以Blob的形式保存的，不同的文件对应不同的Blob，相同的文件对应同一个Blob，如何区分不同的文件呢？只有相同文件名和相同文件内容的文件才能成为相同的文件，例如下图中，Blob1的文件名叫`1.txt`，内容为111，Blob3的文件名也叫`1.txt`，但是内容变成了333，这就是两个不同的Blob。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206251121848.png"  style="zoom: 50%;" />
</div>

对于Commit来讲，同一个对象应当包含相同的`message`,`parents`,`date`,`blobID`,其中`message`是每次执行`gitlet commit`时附带的信息，例如“init commit”，“add file1”等等；`parents`中保存的是前一个Commit的ID信息，当执行完`merge`操作时，会将两个Branch合并，这时候，合并完的Commit会指向两个commit，也就是说这个Commit的`parents`中有两个ID；`date`是执行`commit`时自动生成的时间信息，在这里，一定要注意，gradescope会要求时间信息的格式`Wed Dec 31 16:00:00 1969 -0800`，不是这个格式会在`gitlet log`命令中报错；最后是`blobID`，每个Commit对应多个保存的文件，只需要将文件的`blobID`保存在Commit当中就可以在`objects`文件夹中找到对应的Blob对象了。

总结一下，`objects`文件夹中保存着所有的Commit和文件信息，不管是Commit还是文件，都被称为对象，每个对象都有一个独一无二的ID，所以要想确定一个对象，只需要知道它的ID就可以在`objects`文件夹中找到了。

### refs

`refs`文件夹内存储所有分支的末端信息，在`heads`文件夹内存有多个文件，每个文件的名字即为分支名字，以下图为例，有两个Branch，一个是默认分支`master`，一个是新建立的`61abc`，那么`refs`文件夹的结构即为

``` java
/*

     *      |--refs
     *          |--heads
     *               |--master
     *               |--61abc
     */
```

每个文件的内容是末端Commit的40位ID，`master`文件的内容即为Commit3A的ID，`61abc`文件的内容为Commit3B的ID。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206251027802.png"  style="zoom: 50%;" />
</div>

### HEAD

`HEAD`文件记录的是当前指向的Commit的ID。上图中，`HEAD`文件中的内容即为Commit2的40位ID。

### stage

`stage`为暂存区，当执行`gitlet add "a.txt"`时，只是先将`a.txt`暂存起来，此时`a.txt`的Blob对象已经创建，但是还没有将Commit指向此Blob，执行`commit`后才真正将Commit和Blob连接起来，具体可以参考[这篇文章](https://zhuanlan.zhihu.com/p/96631135)中`add`和`commit`的动图演示，在我的实现版本中，我将`stage`拆开变成`addstage`和`removestage`，分别对应`add`命令和`rm`命令所需要进行的一些操作，这样比较好写代码，小伙伴们可以尝试将这两个写在一起，只需要一个`stage`就可以实现了。:wink:

## 开始前的小建议

**1** 这是一个比较复杂的项目，会写较多的代码与函数，为了让小伙伴们更好的debug以及理解，我建议大伙考虑61a中学到的函数时编程的方法，我说一下我的拙见。

例如为了计算[(2x1)+(2-5)]x[(4x9)/(1+3)]，思路是这样的，先计算第一个方括号，再计算第二个方括号，再相乘。

```java
private int caculate(){
    int a = caculateFirstSquareBrackets();
    int b = caculateSecondSquareBrackets();
    return a*b;
}
```

之后，要计算第一个方括号，就要先计算里面第一个括号，再计算里面第二个括号，最后相加

```java
private int caculateFirstSquareBrackets(){
    int a = caculateFirstBrackets();
    int b = caculateSecondBrackets();
    return a+b;
}

private int caculateFirstBrackets(){
    return 2*1;
}
······
```

类似于这样函数式编程的思想，能够在找bug或者修改功能时快速定位，修改的代码量也会很少，这应该就是61a学到的数据抽象和函数式编程的应用吧，这只是我的理解，大伙看看就好，别较真:joy:。

---

**2**  一定一定一定要写好注释，这样可能大大加快自己找bug的步伐，因为有可能今天写了个比较难的功能，第二天回头找bug发现自己看不懂了，这就很尴尬，大伙还是不要高估自己的记忆力，好记性不如烂笔头，加个注释不会花那么长时间的，此外，养成这样的习惯后，也有利于别人读懂自己的代码，以后不管是参加开源项目还是去了公司，肯定要写注释的，不如现在就先练习起来，为以后打好基础。

## 各功能编写思路

**请各位小伙伴先自己思考，如果有看不明白的地方，再配合此资料以及上面提到的几个进行理解。**:grin:

### 检测args的合法性

如果输入了不正确的`args`或者`args`参数数目不正确，输出错误信息`Incorrect operands.`
### init

```bash
init
```
初始化gitlet仓库，先建立整体的`.gitlet`文件框架，之后建立一个Commit对象initCommit，`message`为`initial commit`，`parents`和`blobID`都为空（注意不是`null`，而是一个空列表，如果为`null`会在产生hashcode环节报错），`date`为`Thursday, 1 January 1970`，直接可以通过以下代码生成

```java
 this.currentTime = new Date(0);
```

此处生成的并不是标准格式的时间戳，可以通过一下代码来转换得到`String`类型的时间戳

```java
 private static String dateToTimeStamp(Date date) {
        DateFormat dateFormat = new SimpleDateFormat("EEE MMM d HH:mm:ss yyyy Z", Locale.US);
        return dateFormat.format(date);
    }
```

最后需要产生ID，可以通过下面代码得到，`sha1()`函数的输入变量必须是`String`类型，所以要将所有变量变为`String`类型才能得到40位ID。

```java
private String generateID() {
        return Utils.sha1(dateToTimeStamp(currentTime), message, parents.toString(), pathToBlobID.toString());
    }

```

这样就建立了一个initCommit对象，之后将其写入`objects`文件夹即可。

**失败的情况**：如果一个`.gitlet`仓库已经存在，输出`A Gitlet version-control system already exists in the current directory.`然后退出。

### add

```bash
add [filename]
```

`add`操作会将所有未存储的文件以Blob的形式存起来，存储的对象别忘了要先实例化,即

```java
public class Commit implements Serializable{...}
public class Blob implements Serializable{...}
```

注意，所有`static`标签标记的变量都不会被存储，所以声明变量时不要加`static`。

除此之外，所有存储起来的文件名以及对应的`blobID`都要写入到`addstage`中，以便于下次`commit`的时候读取。
以下是我的Commit、Blob以及Stage的实例变量。**仅供参考**

```java
/*
 *  commit object
*/
private String message;

private Map<String, String> pathToBlobID = new HashMap<>();

private List<String> parents;

private Date currentTime;

private String id;

private File commitSaveFileName;

private String timeStamp;
```

```java
/*
 *  blob object
*/
private String id;

private byte[] bytes;

private File fileName;

private String filePath;

private File blobSaveFileName;
```

```java
/*
 *  stage object
*/
private Map<String, String> pathToBlobID = new HashMap<>();
```

**失败的情况**：如果文件不存在，输出`File does not exist.`
### commit

```bash
commit [message]
```

实例化一个新的Commit对象并且存起来。具体实现如下：

`message`就是输入的message信息。

`parents`指向前一个Commit，前一个`commitID`可由`HEAD`得到，再通过`commitID`读取相应对象的信息。

`date`直接通过`this.currentTime = new Date();`生成即可，注意格式需要变换一下。

`BlobID`是一个`map`（也可以用`list`，不过不太方便），记录文件路径到文件ID的映射，我的实现方法是先将前一个Commit的`map`复制过来，再将其与`addstage`中的文件比较，得到需要删除的文件以及需要增加的文件，通过`map`的key value操作进行增删。以下图为例，Commit1到Commit2，增加了Blob1和Blob2，Commit2到Commit3删除了Blob1，增加了Blob3。

ID依然通过`sha1`函数生成即可。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206251121848.png"  style="zoom: 50%;" />
</div>

实例化完毕之后，将此Commit保存起来，让`HEAD`指向此Commit，清空`addstage`即完成了`commit`。

**失败的情况**：如果`addstage`中没有内容，输出错误信息`No changes added to the commit.`如果`message`为空，输出错误信息`Please enter a commit message.`

### rm

```bash
rm [filename]
```

`rm`函数功能为删除工作目录中名为filename的文件，这分三种情况：

文件刚被`add`进`addstage`而没有`commit`，直接删除`addstage`中的Blob就可以。

文件被当前Commit追踪并且存在于工作目录中，那么就将及放入`removestage`并且在工作目录中删除此文件。在下次`commit`中进行记录。

文件被当前Commit追踪并且不存在于工作目录中，那么就将及放入`removestage`并即可（`commit`之后手动删除文件，然后执行`rm`，第二次`rm`就对应这种情况）。

除了这三种情况外，工作目录中不可能存在名为filename的文件了，所以`rm`就只对应这三种情况。

**失败的情况**：如果该文件既不在`addstage`区域，也不被当前Commit跟踪，说明工作目录中没有此文件。输出错误信息`No reason to remove the file.`

### log

```bash
log
```

从当前Commit开始倒着打印所有Commit信息，直到initCommit，使用一个循环即可完成此环节。

有一个特殊的Commit叫做`merge commit`，如下图所示，`merge commit`有两个`parents`，打印`merge commit`时只需要选择parentsList中的第一个即可，因为我们在下面`merge`操作时会将所需要打印的`parents`放在List中第一个位置。这里不作详细说明了。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206252219330.png"  style="zoom: 50%;" />
</div>

**失败的情况**：无

### global-log

```bash
global-log
```
打印所有的Commit而不关心顺序，只需要读取`objects`文件夹中的所有Commit，打印出来就好，以下函数功能是返回所有`objects`文件夹中Commit，是以List形式存储的。

```java
List<String> commitList = Utiles.plainFilenamesIn(OBJECT_DIR);
```

其中的`OBJECT_DIR`是是我自定义的，定义如下，这都可以从Lab6中学到，大伙不要傻傻的抄过去发现报错了不知道哪里的问题:joy:。

```java
    public static final File GITLET_DIR = join(CWD, ".gitlet");
    public static final File OBJECT_DIR = join(GITLET_DIR, "objects");
```

**失败的情况**：无

### find

```bash
find [commitmessage]
```

打印所有与输入message相同的Commit的ID，与上面思路类似，先读取所有的Commit，找到符合的，打印出ID即可，如果有多个结果，一行一个。

**失败的情况**：无

### status

```bash
status
```

打印以下信息

```bash
=== Branches ===
*master
other-branch
  
=== Staged Files ===
wug.txt
wug2.txt
  
=== Removed Files ===
goodbye.txt
  
=== Modifications Not Staged For Commit ===
junk.txt (deleted)
wug3.txt (modified)
  
=== Untracked Files ===
random.stuff
```

首先打印所有分支，读取`Heads`文件夹中的文件名即可，当前分支由`HEAD`文件中的内容确定，在前面加`*`号即可。

第二项读取所有保存在`addstage`中的文件名，打印出来即可。

第三项读取所有保存在`removestage`中的文件名，打印出来即可。

后两项是extra credit中的内容，我没有实现，所以只需要打印以下内容即可。

```bash
=== Modifications Not Staged For Commit ===
  
=== Untracked Files ===

```

**失败的情况**：无

### checkout

checkout有三种应用场景

第一种：

```bash
checkout – [filename]
```

当执行完`commit`后，工作目录中存在`1.txt`，我想修改它的内容或者直接删除，更改完或者删除后，我后悔了，就可以调用这第一种命令将文件recover回来。所以思路是这样的，如果当前Commit追踪的文件包含filename，则将其写入到工作目录：如果同名文件存在，overwrite它，如果不存在，直接写入。

**失败的情况**：当前的Commit中不存在filename的文件，输出`File does not exist in that commit.`

---

第二种：

```bash
checkout [commitID] – [filename]
```

与第一种情况类似，只不过现在是从之前的某个Commit追踪的文件中把filename的文件拉过来，只需要去相应的`commitID`执行操作就可以了。

---

第三种：

```bash
checkout [branchname]
```

这种情况下是切换Branch的命令，如下图所示

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206252304468.png"  style="zoom: 50%;" />
</div>

一开始`HEAD`指向的是`master`分支的最新的Commit，当执行`checkout 61abc`后，`HEAD`就指向了`61abc`分支的最新的Commit，并且工作目录中的文件全都会变成Commit3B所包含的Blob文件。那么这个文件转换过程可以分为三种文件：

文件名既被Commit3B追踪的文件，也被Commit3A追踪，那么对于相同文件名但`blobID`不同（也就是内容不同），则用Commit3B种的文件来替代原来的文件；相同文件名并且`blobID`相同，不进行任何操作。

文件名不被Commit3B追踪的文件，而仅被Commit3A追踪，那么直接删除这些文件。

文件名仅被Commit3B追踪的文件，而不被Commit3A追踪，那么直接将这些文件写入到工作目录。这里有个例外，即对于第三种情况，将要直接写入的时候如果有同名文件（例如`1.txt`）已经在工作目录中了，说明工作目录中在执行`checkout`前增加了新的`1.txt`文件而没有`commit`，这时候gitlet不知道是应该保存用户新添加进来的`1.txt`还是把Commit3B中的`1.txt`拿过来overwrite掉，为了避免出现信息丢失，gitlet就会报错，输出`There is an untracked file in the way; delete it, or add and commit it first.`

更改`HEAD`指向Commit3B，最后清空缓存区。

**失败的情况**：如果branchname不存在，输出`No such branch exists.`，如果branchname就是当前分支，输出`No need to checkout the current branch.`


### branch

```bash
branch [branchname]
```

增加一个Branch，即在`heads`文件夹中添加一个新的名为branchname的文件，内容为当前的commitID。此操作不改变`HEAD`指向，只是单纯增加一个Branch。改变分支依然是通过`checkout`命令来改变。

**失败的情况**：无

### rm-branch

```bash
rm-branch [branchname]
```

删除一个Branch，此Branch不能为现在的`HEAD`指向的Branch。具体操作为删除`heads`文件夹中的branchname文件即可。

**失败的情况**：如果给定的branchname不存在，输出`A branch with that name does not exist.`如果尝试删除的Branch为当前Branch，输出错误信息`Cannot remove the current branch.`

### reset

```bash
reset [commitID]
```

此命令即将版本回滚到指定的commitID处，首先将`HEAD`指向指定的commitID，之后进行文件操作，这里的操作其实和`checkout branch`相同，可以复用代码，具体需要小伙伴自己思考，最后清空缓存区即可。

**失败的情况**：如果给定的commitID不存在，输出`No commit with that id exists.`，另外就是类似于checkout的overwrite错误，这里也可能会出现，输出`There is an untracked file in the way; delete it, or add and commit it first.`如果复用checkout的代码，第二种错误实际上是不需要考虑的（因为checkout中代码已经包含了）。

### merge

```bash
merge [branchname]
```

此命令是为了将两个分支进行合并，将branchname与当前分支进行合并，所以branchname一定不能是当前的分支名。`merge`命令分为两步，第一步是找split point，第二步是合并文件。

首先要做的是找到split point，也就是两个分支的最近距离的分开的Commit节点，因为后面要对split point，当前Commit，合并进来的Commit三者进行比较来确定保留的文件，这个之后再说，先看split point。如下图所示

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206260016998.png"  style="zoom: 65%;" />
</div>

图中现有的分支有BranchA，BranchC，BranchMaster，他们的的split point如下表
| Branch1 | Branch2 | Spilt Point|
| ----------- | ----------- |------------|
| BranchA | BranchC |Init commit|
| BranchA | BranchMaster |Init commit|
| BranchC | BranchMaster |commit3|

相信小伙伴们已经了解了split point，接下来说一下怎么寻找这个split point，肯定不可以用暴力两个for循环的方法，因为这个方法时间复杂度是O(n^2)，时间复杂度太高，所以小伙伴们可以先自己考虑以下如何实现。

我的方法是BFS，以BranchA和BranchC为例，先从后往前用BFS算法进行遍历，将所有遍历到的commitID与他们的深度作为key和value放入Map中，例如对于BranchC来说，Commit4B的ID就对应深度1，Commit3的ID对应深度2······，遍历到initCommit停止，这时候，我们有两个Map，一个是BranchC的MapC，一个是BranchA的MapA。现在遍历MapA的keyset，如果MapC也包含此key，记录下对应的MapA的key和value，记为minkey和minvalue。接着遍历下去，对于下一个key，如果MapC也包含此key，并且对应的MapA的value＜minvalue,那么将minkey和minvalue改成现在的key和value，一直到遍历完MapA的keyset，这时候我们记录的minkey和minvalue就对应于split point的ID以及深度。检索完成，时间复杂度为O(n)，这样就找到了split point。

接下来是文件的合并操作。此命令的文件合并有一个[参考视频](https://www.youtube.com/watch?v=JR3OYCMv9b4&t=929s)，可以先了解一下，这个是将`other`合并入`HEAD`分支。

在进行文件合并之前，首先要判断两个类型。

如果split point和`HEAD`分支的Commit相同，意味着`otherbranch`与`HEAD`在一个分支上并且超前于`HEAD`，此时直接将`HEAD`更新到`otherbranch`的当前Commit，并且输出`Current branch fast-forwarded.`

如果split point和`other`分支的ommit相同，意味着`otherbranch`与`HEAD`在一个分支上并且滞后于`HEAD`，此时其实已经完成了`merge`操作，这种情况下什么都不用干，直接输出`Given branch is an ancestor of the current branch.`即可。

视频里的总体思路如下：

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206252341784.png"  style="zoom: 100%;" />
</div>

我举个更通俗易懂的例子，将`61abc`合并入`master`。如下图所示，颜色为白色代表此文件不存在，其他的不同颜色代表内容不同，最终的结果已经画在图中，序号所表示的情况与上图一一对应，例如1中`61abc`文件进行了修改，与splitpoint中不同，颜色就变了，而`master`中则没有修改，最终的结果是新产生的Commit当中保留`61abc`当中的文件。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206260040076.png"  style="zoom: 70%;" />
</div>

也就是说，对于某一个文件，只要是有一个Branch有过修改，就保留这个Branch中对于文件的修改，如果两个Branch都有修改，内容相同，也保留，这些都属于正常情况，产生的新commit message为`Merged [given branch name] into [current branch name].`

如果两个Branch都有修改并且内容不同，那么gitlet不知道保留哪个，就会产生冲突。产生冲突要将先输出`Encountered a merge conflict.`再产生冲突的内容写入到最后的文件中，格式如下

```txt
<<<<<<< HEAD
contents of file in master branch
=======
contents of file in 61abc branch
>>>>>>>
```

我的具体思路是这样的：以`61abc`合并入`master`为例子。先把`master`的Commit放到合并后的Commit位置，再将split point，master，61abc的Commit中的所有文件放入一个ID为key、filename为value的Map中，命名为allfileMap；再分别将上述三个Commit中的文件放入ID为key、filename为value的Map中，分别命名为splitMap，masterMap，61abcMap。遍历allfileMap中的keyset，判断其余三个Map中的文件存在以及修改情况，就能够判断出上述7种不同情况，然后对每个文件进行删除、覆写、直接写入等操作，这样就完成了`merge`操作。

**失败的情况**：如果缓存区存在文件，输出`You have uncommitted changes.`如果给定的Branch不存在，输出`A branch with that name does not exist.`如果给定的Branch和当前Branch相同，输出`Cannot merge a branch with itself.`如果工作目录存在仅被merge commit跟踪，且将被覆写的文件，输出`There is an untracked file in the way; delete it, or add and commit it first.`

`merge`操作理解起来并不难，主要是情况分类有点多，容易出现错误，我在`merge`这里花了大概十几个小时，大部分都在调试，bug都是很小的错误，但是调试起来比较麻烦，所以小伙伴们一定要有耐心，慢慢来，最后肯定能够调试出来的！

到这里就可以恭喜你已经完成gitlet了！！！！

**完结撒花**！！！:cherry_blossom::tulip::hibiscus::rose::sunflower::bouquet:

## 调试指南

由于gradescope每20分钟充能一次，并且输出的`.gitlet`是不可见的，所以调试起来非常麻烦，我在这里提供本地测试用例，但此测试会有一定的bug，test17，33，36，43不可用，最终结果请以gradescope为准。

请先下载[testingFile](https://github.com/Yukang-LIAN/Gitlet-Testing-Files)，这里感谢null哥爬取了gitlet的测试用例，之前只能够支持20几个test，我在这个基础上补充了一些文件，现在本地测试能够支持30几个test，具体用法如图所示

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206261113659.png"  style="zoom: 65%;" />
</div>

将`testing`文件夹放到`proj2`的目录中，进入`testing`文件夹

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206261116150.png"  style="zoom: 65%;" />
</div>

所有的测试用例都在`samples`文件夹下，要想运行test01，在`testing`文件夹中打开`git bash`中输入`python3 tester.py samples/test01-init.in`，要想获得详细的输出，修改以上命令为`python3 tester.py --verbose samples/test01-init.in`，要想一次性运行所有测试，输入`python3 tester.py samples/*.in`。所有在测试过程中产生的`.gitlet`文件夹都会在`testing`文件夹中出现，可以自行查找问题，如果本地和gradescope出现矛盾的输出，请以gradescope为准。

注意：运行前一定要先在gitlet源文件目录下运行`javac *.java`编译文件才可以，有时候修改了bug但发现跑不过测试，请先检查有没有编译新的java文件，如果没有编译那么运行的依旧是旧版本。

## 踩过的坑

在找split point时，一开始没有考虑太多情况，直接当成是两个链表找最近的公共节点来做了，实际上的Branch可能有多个，所以这方面没有考虑完全，调试找bug花了一些时间。

`merge`命令的conflict环节，我先判断了会不会产生conflict，产生后写入到文件中，后进行文件合并，导致我写入的包含有conflict内容的文件在之后被删除了，跑测试的时候一直报错说文件内容不正确，我就一直在找哪里内容不正确，找了四五个小时，一开始其实本地测试也不显示`.gitlet`文件夹，我也不知道哪里有问题，最后我修改了`tester.py`文件将`.gitlet`文件夹显示出来，才发现不是内容不正确，是根本就没这个文件:joy:。我提供给大家的测试文件都是修改后可以显示`.gitlet`文件夹的，所以可以让大家节省很大一部分时间，不要再走我走过的弯路。

## 总结

gitlet是一个1000行左右的项目，体量并不是很大，但是包含的知识点和思想很丰富，这个项目可以充分的锻炼入门的新手来实现一个刚刚好的项目，我做完这个项目收获很大，比如自己对于函数的命名，面向对象思想的理解，函数式编程的方法，以及归纳整理的能力都得到了提高，在这里感谢UCB提供了这样一个平台，感谢josh老师，感谢61abc群友们的帮助，希望与你们一起成长！

BTW，助教哥说过好多次，这个项目是可以放进简历里面的，所以请各位小伙伴尽量自己实现，这也是以后和面试官吹牛逼的资本:yum:。

完。

<div align="center">
    <img src="https://cdn.jsdelivr.net/gh/Yukang-LIAN/PictureRepo/202206261201031.jpg"  style="zoom: 80%;" />
</div>