## 简介
Blog主题使用[NexT](https://github.com/iissnan/hexo-theme-next)

## 参考命令
+ 添加与获取submodule的命令如下，submodule其它操作可以参考“[Git Submodule使用完整教程](http://www.kafeitu.me/git/2012/03/27/git-submodule.html)”
 ```
 git submodule add https://github.com/iissnan/hexo-theme-next.git themes/next

 git submodule init
 git submodule update
 ```

+ 远程编辑库使用方式（src：dst）
 ```
 git remote add -t dev edit https://github.com/stillwuyan/stillwuyan.github.io.git
 git push edit master:dev
 git pull edit dev:master
 ```
