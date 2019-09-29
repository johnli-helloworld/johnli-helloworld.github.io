## git base 命令使用

- [1 git设置默认编辑为vim](#git设置默认编辑为vim)

- [2 git修改已提交的记录的用户名邮箱信息](#git修改已提交的记录的用户名邮箱信息)
  - [2.1 修改某次提交的用户信息](#修改某次提交的用户信息)
  - [2.2 批量修改提交的用户信息](#批量修改提交的用户信息)
- 3 [合并多次提交记录](#合并多次提交记录)




## 1 git设置默认编辑为vim
``` sh
export GIT_EDITOR=vim
```

## 2. git修改已提交的记录的用户名邮箱信息

### 2.1 修改某次提交的用户信息

- 第一步，使用git log命令查看版本记录，选取你需要修改的某次提交的后一个提交的SHA1 ID值（如我需要修改Author: john的这次提交的用户名和邮箱，则我需要选取下面列SHA1 ID为`010da6f5` 印象八位即可）， git log 显示如下:

```sh
# git log
commit 5f5d2a847f5a134e94fe5113b51094193113f510
Author: way <way.w.y@foxmail.com>
Date:   Fri Sep 20 16:14:15 2019 +0800

    modify go get

commit 949ee403b3895f634ec6e704fc8cb4f752abde33
Author: john <ll@126.com>
Date:   Thu Aug 29 17:32:02 2019 +0800

    go-filecoin api v0.0.1

commit 010da6f5dca2672189e4064d0bb01a706713243f
Author: way <way.w.y@foxmail.com>
Date:   Thu Aug 22 15:44:35 2019 +0800
```

- 第二步，回到命令，开始执行rebase  -i 操作， 如下:

```sh
# git rebase -i 010da6f5
```

这个时候git会自动打开一个界面，如下:

```
pick 949ee40 go-filecoin api v0.0.1
pick 5f5d2a8 modify go get
# Rebase ...
#
# Commands:
...
```

- 第三步，修改第一行数据（就是我们预期修改的那条commit）的pick为edit，如下：

```
edit 949ee40 go-filecoin api v0.0.1
pick 5f5d2a8 modify go get
# Rebase ...
#
# Commands:
...
```

保存退出，可以看见如下：

```
Stopped at xxxx ... go-filecoin api v0.0.1
You can amend the commit now, with

	git commit --amend 

Once you are satisfied with your changes, run

	git rebase --continue
```

这时候就可以通过git commit -amend来修改用户名和邮箱了，操作如下：

```sh
# git commit --amend --author="new-name <new-email@qq.com>" --no-edit
```

继续完成`rebase`，

```sh
# git rebase --continue
```

最后，通过`git push --force`将篡改历史纪录后的结果同步到服务器



### 2.2 批量修改提交的用户信息

新建脚本文件modify-userinfo.sh，内容如下：

```sh
git filter-branch --force --env-filter '
  #如果Git用户名等于老的Git用户名 wangshuyin
  if [ "$GIT_COMMITTER_NAME" = "<Old Name>" || "$GIT_AUTHOR_EMAIL" = "<Old Email>" ];
  then
    #替换提交的用户名为新的用户名，替换提交的邮箱为正确的邮箱
    GIT_COMMITTER_NAME="<New name>";
    GIT_COMMITTER_EMAIL="<New email>";
    
    #替换用户名为新的用户名，替换邮箱为正确的邮箱
    GIT_AUTHOR_NAME="<New name>";
    GIT_AUTHOR_EMAIL="<New email>";
  fi
' --tag-name-filter cat -- --all
```



## 合并多次提交记录

如下，我想合并最新的两次请求：

```sh
# git log
commit ad61014300d3ee8b33ccb0f3bfe3c557b7c07ad7
Author: john <ll5@126.com>
Date:   Thu Sep 26 17:12:28 2019 +0800

    sssssssssss

commit dbfff5d853d912762e93b8eb79c504e3f353074c
Author: john <ll@126.com>
Date:   Thu Sep 26 17:11:42 2019 +0800

    ddddddddddd

commit 226d1882a6cbc9ce70919b40f8d5bb446a9c811c
Author: john <ll@126.com>
Date:   Thu Sep 26 15:25:58 2019 +0800

    test
```

- 第一步，执行如下命令：

```sh
# git rebase -i 226d1882a
```

这时会自动打开一个界面：

```
pick dbfff5d ddddddddddd
pick ad61014 sssssssssss

# Rebase 226d188..ad61014 onto 226d188 (2 command(s))
#
# Commands:
...
```

修改第二行数据的pick为s（s为squash的缩写）：

```
pick dbfff5d ddddddddddd
s    ad61014 sssssssssss

# Rebase 226d188..ad61014 onto 226d188 (2 command(s))
#
# Commands:
...
```

按esc 保存退出，会马上进入另一个界面：

```
# This is a combination of 2 commits.
# The first commit's message is:

ddddddddddd

# This is the 2nd commit message:

sssssssssss

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Date:      Thu Sep 26 17:11:42 2019 +0800
#
# interactive rebase in progress; onto 226d188
# Last commands done (2 commands done):
#    pick dbfff5d ddddddddddd
#    s ad61014 sssssssssss
# No commands remaining.
# You are currently editing a commit while rebasing branch 'master' on '226d188'.
#
# Changes to be committed:
#       modified:   filename
```

这时候我们可以填写想要的commit message，例如我把第二个message `sssssssssss`清空，则会合并为一个commit message：

```
# This is a combination of 2 commits.
# The first commit's message is:

ddddddddddd

# This is the 2nd commit message:


# Please enter the commit message for your changes. Lines starting
...
```

按esc保存退出，可以看到成功的信息，最后，通过`git push --force`将篡改历史纪录后的结果同步到服务器。此时我们可以看到git log的记录只有`dddddddddd`了：

```sh
# git log
commit 888233818a79f2fa2bcfcd92833a0927c3ea69c1
Author: johnli-helloworld <ll19940705@126.com>
Date:   Thu Sep 26 17:11:42 2019 +0800

    ddddddddddd

commit 226d1882a6cbc9ce70919b40f8d5bb446a9c811c
Author: johnli-helloworld <ll19940705@126.com>
Date:   Thu Sep 26 15:25:58 2019 +0800

    test
```