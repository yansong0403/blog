介绍
====

使用Sphinx + GitHub +
ReadtheDocs搭建个人博客，Sphinx用来生成html文档，GitHub用来托管文档，最后再导入到ReadtheDocs中。

1. 安装sphinx
=============

Sphinx是一个基于Python的文档生成工具，我使用的python版本是2.7。安装Sphinx及相关工具的命令为：
``pip install sphinx sphinx-autobuild sphinx_rtd_theme``

2. 生成工程
===========

1. 创建工程目录 ``mkdir Blog``
2. 生成工程 ``cd Blog``\ ，在该目录下执行\ ``sphinx-quickstart``
3. 配置工程，将生成的’Blog/source/conf.py’文件中的\ ``html_theme = 'alabaster'``\ 替换为我们之前下载的主题\ ``html_theme = 'sphinx_rtd_theme'``
4. 生成网页，在’Blog’文件夹下面执行\ ``make html``

3. 托管到GitHub上
=================

::

   git init
   git add .
   git commit -m "sphinx start"
   git remote add origin https://github.com/[yourname]/[yourrepository].git
   git push origin master

4. 发布到ReadtheDocs上
======================

注册ReadtheDocs并关联GitHub账号，导入代码库，填好对应的信息，便可以生成在线地址

5. 构建文档结构
===============

在source/index.rst文件中定义了文档结构，在执行\ ``make html``\ 之后，会根据该文件生成html。更改index.rst文件内容为：

::

   目录
   ====
   .. toctree::
       :maxdepth: 2
       :caption: Contents:
       
       BuildBlog/index

BuildBlog/index.rst文件中的内容为：

::

   搭建个人博客
   ============
   .. toctree::
       :maxdepth: 2

       content

content.rst为文档的真正的内容。

6. 撰写文档
===========

sphinx使用自动生成的文档是rst格式，这种格式文件可以使用pandoc从markdown文件转换得到，对于不想学习rst语法的同学可以直接撰写markdown文件，再使用转换命令得到rst文件。
``pandoc content.md -o content.rst``
