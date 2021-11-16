# [返回](/)

# Git基本命令

**上传一个本地的独立分支**
1、git init （初始化环境，生成.git环境）
2、git add . （讲文件添加暂存区）
3、git commit -m “描述信息” （提交暂存区文件）
4、git branch 分支名称 （创建本地分支）
5、git checkout 分支名称 （切换到本地分支）
6、git remote add origin 远程仓库地址 （关联远程仓库）
7、git push origin 分支名 （推送本地分支到远程仓库）

**上传本地仓库到远程仓库指定分支**
1、创建本地文件夹，并在次文件夹处打开Git Bash
2、git init （初始化环境，生成.git环境）
3、git remote add origin 远程仓库地址 （连接远程仓库）
4、git pull origin 远程仓库分支名 （拉取远程仓库指定分支）
5、git checkout --track origin/远程仓库分支名 （追踪远程仓库分支，与本地仓库建立联系）
6、在本地仓库中添加文件或修改文件
7、git add . (讲文件添加暂存区)
8、git commit -m ‘描述信息’ （提交暂存区文件）
9、git push origin 远程仓库分支名 (推送到远程仓库指定分支)

中间可能会需要输入个人信息

```bash
git config --global user.name "xxx"
git config --global user.email "xxx@email.com"
```

# 错误解决

## Updates were rejected because the tip of your current branch is behind

> [https://www.cnblogs.com/651434092qq/p/11015806.html——咸咸海风](https://www.cnblogs.com/651434092qq/p/11015806.html)

**1. 使用强制push的方法：**
git push -u origin master -f
这样会使远程修改丢失，一般是不可取的，尤其是多人协作开发的时候

**2.push前先将远程repository修改pull下来**
git pull origin master
git push -u origin master

**3.若不想merge远程和本地修改，可以先创建新的分支：**
git branch [name]
然后push
git push -u origin [name]

## Support for password authentication was removed on August 13, 2021. Please use a personal access token instead

> [https://www.jianshu.com/p/3b4f7619999e——EllisonPei](https://www.jianshu.com/p/3b4f7619999e)

1. 创建自己的token

2. 打开.git/config，修改文件

   ```text
   [remote "origin"]
       url = https://ellisonpei:这里填access_token@github.com/项目名/项目名.git
       fetch = +refs/heads/*:refs/remotes/origin/*
   ```



# 参考

- [https://www.cnblogs.com/651434092qq/p/11015806.html——咸咸海风](https://www.cnblogs.com/651434092qq/p/11015806.html)
- [https://www.jianshu.com/p/3b4f7619999e——EllisonPei](https://www.jianshu.com/p/3b4f7619999e)

