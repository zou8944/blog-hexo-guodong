# 果冻的个人博客 - 基于Hexo
我是果冻，这是我的个人博客。基于Hexo日志框架，安装了next主题。
# 前置条件
如果需要运行本项目， 请确保已安装nodejs，git，docker，hexo-cli，hexo-server。如未安装，请参考如下方式。
- nodejs
- git
- docker
- hexo-cli
- hexo-server 
# 启动
在本机的80端口启动
```bash
hexo server -p 80
```
但实际上为了能够支持http访问, 采用nginx配置.

原理是利用hexo generate生成静态页面, 利用nginx访问静态页面

# 修改配置

配置可以在根目录的_config.yml中修改，也可以在主题目录的_config.yml中修改。几乎所有的网站配置都在该配置文件中，包括网站标题、作者信息、各种插件等内容，使用起来也是非常方便的。
