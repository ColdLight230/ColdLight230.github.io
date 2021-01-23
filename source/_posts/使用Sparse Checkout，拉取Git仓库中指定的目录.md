---
title: 使用Sparse Checkout，拉取Git仓库中指定的目录
date: 2020-09-11 
tags: git技巧
categories: git

---

某些git仓库经历了N多个版本，迭代。有可能仓库特别大，而开发新功能可能只需要该仓库中的某些文件夹。那么就可以使用`sparse checkout`拉取仓库中指定文件。

<!--more-->

### 未拉取仓库的情况
```
$mkdir project_folder
$cd project_folder
$git init
$git remote add -f origin <url>
```
与远程项目关联上

如果已经拉取过直接走下面的逻辑

### 配置sparsecheckout
接下来进入仓库目录，在`Config`中允许使用`Sparse Checkout`模式
```
git config core.sparsecheckout true
```

### 配置指定目录
编辑` .git/info/sparse-checkout `若没有此文件，需要手动创建
1. 可以添加文件名称指定拉取文件夹
```
product
```
2. 也可以通过`!`设置排除的文件夹
```
*
!Chromium/**
!ChromiumRes/**
```

### 如果需要添加目录，就增加`sparse-checkout`的配置，再`checkout master`
```bash
echo another_folder >> .git/info/sparse-checkout
git checkout master
```

