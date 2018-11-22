## 简介
个人博客，主题使用[Maupassant](https://github.com/tufu9441/maupassant-hexo)

## Hexo使用说明

### 初始化Blog

+ 安装Hexo CLI后，需要创建基础的Blog目录结构并安装依赖

  ```
  npm install hexo-cli -g
  hexo init blog
  cd blog
  npm install
  hexo server
  ```

### 恢复Blog工作区

+ 从github下载Blog工作目录后，需要安装Hexo CLI和相关依赖

  ```
  git clone https://github.com/stillwuyan/stillwuyan.github.io.git blog
  cd blog
  git checkout -b edit origin/dev
  npm install hexo-cli -g
  npm install
  ```

+ 移除NexT主题并安装Maupassant主题

  ```
  git submodule deinit -f themes/next
  rm -rf .git/modules/themes/next		//如果没有下载过modules，该路径可能不存在
  git rm themes/next
  git submodule add https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
  npm install hexo-renderer-pug --save
  npm install hexo-renderer-pug --save
  ```

  > 注意需要修改`_config.yml`的`theme`值为`maupassant`

## 工作区说明

### 编辑

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

### 部署

+ 将静态文件上传至git库，通过[`https://stillwuyan.github.io/`](https://stillwuyan.github.io/)可以访问页面。

  ```
  hexo deploy -g
  ```


### 保存工作区

+ 远程编辑库使用方式（src：dst）
 ```
 git remote add -t dev edit https://github.com/stillwuyan/stillwuyan.github.io.git
 git push edit master:dev
 git pull edit dev:master
 ```



## 参考链接

1. [如何创建分类页面](https://github.com/iissnan/hexo-theme-next/wiki/%E5%88%9B%E5%BB%BA%E5%88%86%E7%B1%BB%E9%A1%B5%E9%9D%A2)