# [返回](/)

# Git基本命令



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

