### 1、初始化仓库  
> repo init -u  url   

### 2、同步代码  
> repo sync -c -jx  

### 3、新同步代码所有git仓切换分支  
> repo start branchname --all  


### 4、清空git仓  
> repo forall -c 'git reset --hard HEAD;git clean -df'  
> 

### 5、提交服务器  
```
git commit --amend -s

git push origin HEAD:refs/for/O-M01-PUBLIC
或
repo upload .
```

### 6、删除分支  
> repo abandon branchname  
> 

### 7、查看分之  
> repo branch  或 repo branches  