---
title: AppleDoc
categories: 工具
---


[iOS 开发_编写接口文档（appledoc实用篇）](https://www.jianshu.com/p/f5cb3c728a5b)
[appledoc自动输出开发文档教程](https://blog.csdn.net/xborong/article/details/85057696)

```
git clone git://github.com/tomaz/appledoc.git
cd ./appledoc
sudo sh install-appledoc.sh
```

脚本

```
#!/bin/bash
appledoc \
#文档输出目录
--output ./apiDoc \                                       
#忽略.m文件，因.m中均为私有api和属性，开源的接口文档中理应忽略掉
-i *.m \                                                         
#工程的名字
--project-name "testAppledocDemo" \
#公司的名字
--project-company "Zyp" \
#不生成docset，直接输出html
--no-create-docset \
#没有注释的文件也输出html  -->目的是看到所有的文件
--keep-undocumented-objects \
#没有注释的属性和方法也输出到html  -->目的是看到所有的属性和方法
--keep-undocumented-members \
#没有注释的文件不提示警告
--no-warn-undocumented-object \
#没有注释的属性和方法不提示警告
--no-warn-undocumented-member \
#需要输出的文件路径  -->这里推荐最好直接为当前工程路径平级输出，便于维护和使用
./
```

注意：需要将注释删除，不然会报错

```
Caught: AppledocException: At least one directory or file name path is required, use 'appledoc --help'
./myProDoc.sh: line 4: --output: command not found
./myProDoc.sh: line 6: -i: command not found
./myProDoc.sh: line 8: --project-name: command not found
./myProDoc.sh: line 10: --project-company: command not found
./myProDoc.sh: line 13: --no-create-docset: command not found
./myProDoc.sh: line 15: --keep-undocumented-objects: command not found
./myProDoc.sh: line 17: --keep-undocumented-members: command not found
./myProDoc.sh: line 19: --no-warn-undocumented-object: command not found
./myProDoc.sh: line 21: --no-warn-undocumented-member: command not found
./myProDoc.sh: line 22: ./: is a directory
```

权限错误

```
-bash: ./myProDoc.sh: Permission denied
```

获取脚本权限

```
chmod +x myProDoc.sh
```

```
 @brief         --> 简要描述
 @param         --> 用于参数说明
 @see           --> 可见的链接性说明，文档中可对应链接到内容 一般可用于注释枚举属性的类型
 @discussion    --> 详细说明 提醒信息
 @warning       --> 警告内容
 @bug           --> bug内容
 @return        --> 返回值说明
```