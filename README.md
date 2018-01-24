## 简介
Blog主题使用[NexT](https://github.com/iissnan/hexo-theme-next)



## 工作区说明

### 1. 编辑

+ 新建文章，文章名称建议用英文，单词间用`-`分隔。

  ```
  hexo new post "article-name"
  INFO  Created: D:\myblog\source\_posts\2017-10-19-article-name.md
  ```

+ 预览文章效果，网站会在`http://localhost:4000`下启动。服务器启动期间，Hexo将监视文件变更并更新。

  ```
  hexo server
  ```

+ 生成静态文件，文件生成在`public`文件夹下。

  ```
  hexo generate
  ```

  > **说明：** 由于部署时也会重新生成静态文件，因此该过程可以省略。

### 2. 部署

+ 将静态文件上传至git库，通过[`https://stillwuyan.github.io/`](https://stillwuyan.github.io/)可以访问页面。

  ```
  hexo deploy -g
  ```


### 3. 保存工作区

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



## 参考链接

1. [如何创建分类页面](https://github.com/iissnan/hexo-theme-next/wiki/%E5%88%9B%E5%BB%BA%E5%88%86%E7%B1%BB%E9%A1%B5%E9%9D%A2)