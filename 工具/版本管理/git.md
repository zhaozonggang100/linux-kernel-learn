### 1、GIT daemon建立本地仓库

- 1、本地仓库根目录执行如下命令，表示可以从本地clone

```
git daemon --verbose --export-all --base-path=.
```

- 2、局域网其他用户执行如下命令clone

```
git clone git://$hostip
```

### 2、git status 显示中文和解决中文乱码

`git config --global core.quotepath false`