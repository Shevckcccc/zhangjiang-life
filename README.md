# 浦东张江wiki
##怎么提交代码
先fork，再提pull request

##格式
####分类/文件名格式
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
在_post目录下有各个分类目录，在对应分类下新建文章(比如租房就是house目录，如果没有分类目录，需要自己新建一个)。然后创建文章的md文件，格式为yyyy-mm-dd-title.md （eg: 2014-10-25-how-to-apply-xxx.md），注意文件名不要跟_post里的任一文件重复。

####内容格式
```
---
layout: post
title: 开篇
category: 租房
authors:
- Shevckcccc|iOS开发|/images/author-image.jpeg|http://java.im
- Will|Java程序员，SAP|http://sameple.com/avatar|

---
```
layout: 文章模块统一post			
title: 标题				 
category: 分类(**注意分类文件夹仅作物理识别用，程序只认此处的category，所以此处要和当前所在的目录文件夹对应，如house目录，这里就是租房**)    
author: 作者数组，首发文章只填一个就行了，多人合作编辑的请依次填写，使用"|"分隔，第一个字段名称，第二个字段描述，第三个字段头像，第四个字段是你的个人网址







