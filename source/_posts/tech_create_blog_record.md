---
layout: post
title: "通过Hexo自建博客"
date: 3/12/2017 6:41:32 PM 
comments: true
tags: 
	- 技术 
	- 博客搭建
---
---
【作为一名技术开发人员，搭建一个属于自己的博客，记录自己，沉淀自己，还是很重要的。】

年前给自己定一个目标：搭建博客，记录成长。之后，你懂的，就有下面的内容：

# 一、Hexo介绍、安装及实操
[Hexo平台搭建官网](https://hexo.io/zh-cn/docs/index.html)

通过hexo官网步骤，可以很快的搭建hexo平台。通过命令hexo s,然后，在浏览器中输入http://localhost:4000/ 网站显示如下：
![](/assets/img/tech_create_blog_record_img01.png)

# 二、Hexo主题选择
通过Hexo官网，搭建的博客使用的主题为hexo自带的主题landscape,效果如上图。如果你希望你的博客更炫，更酷，推荐使用第三方的Hexo主题。主题选择：
<!-- more -->
1.[Next主题](http://theme-next.iissnan.com/)

2.[Hexo主题集合(github)](https://github.com/hexojs/hexo/wiki/Themes)

3.[Hexo主题集合(知乎)](https://www.zhihu.com/question/24422335)

**主题配置：**

```
在hexo配置文件_config.yml中修改 theme: 主题名字（如：theme: landscape)
```

**非Hexo主题(搭建网站可用)**

[jekyll主题集合(官网)](http://jekyllthemes.org/)

[jekyll主题集合(github)](https://github.com/jekyll/jekyll/wiki/Sites)

[WordPress主题集合](https://wordpress.org/themes/)

# 三、部署到Github上
 [Github官网](https://github.com/)

1.注册Github账号

2.创建Repository（格式为：github账号名.github.io）

```
注意Repository的名字必须和账号名相同。比如Github账号是angelen10，那么应该创建的Repository的名字是：angelen10.github.io。
```

3.在hexo配置文件_config.yml中配置git路径

```
xxxx为github账户名
deploy: 
  type: git 
  repository: git@github.com:xxxxx/xxxx.github.io.git
  branch: master
```

4.设置SSH keys绑定到你建的repository

[设置SSH key官方教程](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/)

5.Hexo基本操作命令

```

hexo g == hexo generate

hexo d == hexo deploy

hexo s == hexo server

hexo n == hexo new

```

# 四、相关链接

[搭建Hexo博客并部署到Github详细教程](http://blog.sina.com.cn/s/blog_4c44643f0102vuju.html)

[如何搭建一个独立博客——简明Github Pages与Hexo教程](http://www.jianshu.com/p/05289a4bc8b2)

[利用Github Page 搭建个人博客网站](http://blog.csdn.net/tzs_1041218129/article/details/53214497)









