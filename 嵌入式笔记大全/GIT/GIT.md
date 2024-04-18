# 1.新建仓库

```git
1、git init /*在某个文件夹下使用，创建仓库*/
2、git clone + 仓库地址
```

# 2.工作区域和文件状态

git ls-files 查看暂存区的内容

![image-20240415204256608](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\GIT\GIT.assets\image-20240415204256608.png)

![image-20240415204319469](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\GIT\GIT.assets\image-20240415204319469.png)

# 3.添加和提交文件

**1、git status** /*查看当前仓库处在什么样一个分支，有哪些文件和哪些文件状态*/

**2、git add**    /*把文件添加到暂存区，等待后续的操作*/

**3、git commit** /*提交暂存区的文件到仓库中，而不会提交工作区的文件，一般用法为git commit -m "这里是提交记录的说明，比如你对这个文件干了啥"*/

**4、git reset**的三种模式：

​	git reset --soft 表示回退到某一个版本，并且保存工作区和暂存区中的所有修改内容

​	git reset --hard 表示回退到某一个版本，并且丢弃工作区和暂存区中的所有修改内容

​	git reset --hard 表示回退到某一个版本，并且保留工作区和丢弃暂存区中的所有修改内容

![image-20240415205302417](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\GIT\GIT.assets\image-20240415205302417.png)

5、git log --online 简易查看git操作历史

![image-20240415210106211](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\GIT\GIT.assets\image-20240415210106211.png)

6、git diff

HEAD指向最新的一个分支提交节点

第一种情况，后面不跟任何东西，那么就表示只显示工作区和暂存区的不同。

第二种，比较工作区和版本库的内容，git diff HEAD

第三种，git diff --cached 比较暂存区和版本库的差异

第四种，git diff +两个版本的提交ID就可以查看两个版本之间的差异内容

第五种，如果想用最新的和上一个版本作比较，则是git diff HEAD~ HEAD或git diff HEAD^ HEAD 在此git diff HEAD~2 HEAD表示HEAD之前的两个版本

# 4.删除文件

git rm加上要删除的文件名，可以把文件从工作区和暂存区中删除，删除后切记别忘了提交

也可以先删除文件，然后git add一下

如果只想删除版本库里的，而不想删除本地的，则可以git rm --cached

# 5.gitignore忽略文件

可以vim打开.gitignore文件，然后在里面加上一行*.log表示忽略所有log文件，然后切记wq保存退出

![image-20240416210751343](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\GIT\GIT.assets\image-20240416210751343.png)

![image-20240416210857511](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\GIT\GIT.assets\image-20240416210857511.png)



# 6.克隆GIT HUB仓库

1、一般采用HHS方式，先创建一个新的仓库，然后点击CODE，复制下拉的HHS链接

![msedge_8ayD4zc5dU](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\GIT\GIT.assets\msedge_8ayD4zc5dU.png)

2、在所想要创建仓库的地方打开git bash，创建一个名为.ssh的文件夹，cd进入.ssh此文件夹

输入ssh-keygen -t rsa -b 4096 回车，输入你的秘钥名称以后，然后就是要你输入你的密码。输入两次密码后可以ls -ltr查看一下当前文件夹，会有两个文件，一个是公钥（.pub结尾），一个是私钥（不能给或者上传到git hub）

3、打开.pub结尾的文件，复制里面的内容，在GitHub官网上找到添加SSH秘钥的地方，粘贴进去，然后命名一下即可。

4、如果是这个.ssh是新的，那么还要在当前的.ssh目录下新建一个config文件，里面需要有以下几行：

```
# github
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ./Embedded notebook
```

表示在访问github.com的时候，指定使用当前目录下的Embedded notebook这个秘钥，然后就可以回到本地仓库执行git clone + 复制的SSH路径了，这里会要你输入密码，就是创建SSH秘钥的时候输入的密码

# 7.关联本地仓库和远程仓库

1. 首先还是需要在GitHub上创建一个新的仓库
2. 打开git bash，切换到本地仓库的位置
3. 添加远程仓库的步骤如下

```
1.git remote add origin git@github.com:sasasasa/Embedded-notebook.git /*这个origin就是这个远程仓库的别名，一般默认都是这个，也可以指定为其他，后面就是origin对应的远程仓库的地址*/
2.git branch -M main    /*指定默认的分支为main，如果不是main则要修改*/
3.git push -u origin main /*把本地的main分支和远程仓库origin的main分支关联起来，其实全名应该是git push -u origin main:main，意思就是吧本地的main分支推送给远程的main分支，如果两个分支名称相同则可以改为：git push -u origin main*/
```



















































