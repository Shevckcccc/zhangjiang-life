# 浦东张江生活文档1.0版

### 如何安装

1.请自行网上搜索安装jekyll，如果你已有ruby，输入以下代码即可
```
    $ gem install jekyll
```



2.运行
```
$ git clone git@github.com:Shevckcccc/zhangjiang-life.git
$ cd 211.im
$ jekyll --serve
```



3.访问 http://localhost:4000

### 怎么提交代码
先fork，再提pull request

### 格式

#### 分类/文件名格式
```
文件结构
_posts
   |--house
   		|--2014-11-02-start.md
   |--life
   		|--2014-11-02-buy.md
   |--ohter
   		|--2014-11-02-how-to-apply-xxx.md
   ...

```
请在对应的分类目录下新建文章(如没有分类则新建一文件夹），文件格式为 yyyy-mm-dd-xxx.md

#### 内容格式
```
---
layout: post
title: 开篇
category: 租房
authors:
- Shevckcccc|iOS开发|/images/author-image.jpeg|http://java.im
- Will|Java程序员，SAP|http://sample.com/avatar|

---
```
**layout**: 文章模块统一为post                
**title**: 标题                   
**category**: 分类(`注意分类文件夹仅作物理识别用，程序只认此处的category，所以当前category要和当前所在的目录文件夹一一对应，如在house目录下添加文章，那category就是租房`)                
**authors**: 作者数组，首发文章只填一个就行了，多人合作编辑的请依次填写。作者字段采用"|"分隔，第一个字段名称，第二个字段描述，第三个字段头像，第四个字段是你的个人网址                






