## git命令行中文乱码

&emsp;&emsp;命令git status等遇到中文文件时，可能显示乱码，例如：
```c
ws@ubuntu:~/Src/linux_learning$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   "ubuntu\346\227\240\347\275\221\350\267\257\350\247\243\345\206\263\346\226\271\346\241\210.md"

no changes added to commit (use "git add" and/or "git commit -a")
```

&emsp;&emsp;其正确显示如下：

```c
ws@ubuntu:~/Src/linux_learning$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   ubuntu无网路解决方案.md

no changes added to commit (use "git add" and/or "git commit -a")
```

## 解决办法

&emsp;&emsp;对于git status命令的中文显示乱码问题，可以通过以下两种方式解决：

**1.命令行设置：**
```c
git config --global core.quotepath false
```
&emsp;&emsp;将git配置文件.gitconfig的core.quotepath项设置为false。

**2.手动修改**
&emsp;&emsp;手动将home目录下的.gitconfig文件的quotepath属性修改为false：
```c
ws@ubuntu:~/Src/linux_learning$ cat ~/.gitconfig 
[user]
        email = 18280296472@163.com
        name = ws
[core]
        quotepath = false
```
