---
title: "wanb"
description: "work"
date: "2025-06-03"
tags:
  - 工作
---
✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨✨
## 公共方法
``` java
PartUtils.getPartByNumber  //根据编号获取最新版本部件
IBAUtils.getIBAStringValue // 获取IBA的值
IBAUtils.setIBAStringValue //设置 IBA
IBAUtils.updateIBAStringValues //更改多个 IBA 的值
IBAUtils.getIBAValues //获取 IBA 多个值
queryMpmProcessPlanByNumber //查询最新版本工序
MPMStandardCCUtil.createCC //创建控制特征
SessionServerHelper.manager.setAccessEnforced(false);// 权限问题
LifeCycleUtil.reSetLifeCycle //重新设置生命周期
findDocByNumber //根据编号查询文档
getAllChildPart //获取所有最新子件
newWTChangeOrder// 创建变更通告
//获取当前产品的工艺负责人
//获取角色下的所有用户
//根据promotionNotice对象获取签审列表
//<!--特定替代表格-->
MPMBaseBean.java
根据子件获取上一层父件集合 
getSameSeqParts //根据部件编号获取相同产品序列下的其他M视图研发物料
 该料号不存在
location = FolderHelper.service.saveFolderPath(docFolder, containerRef);// 若文件夹不存在，则创建该文件夹
工艺编制任务表格
Debug.P(String.format("变更物料编号: %s%s", part.getIdentity(), state));//日志打印
getPartSubstituteRelate//获取替代关系
StrategyEnum.MPE_INSERT_OPERATION.getName()// 增加工序
addMESOperationAttribute //MES新增工序 自动增加属性
```
## 项目

### 日志输出
log4j

输出规范

增加：
```java
private static final Logger logger = LogR.getLogger(OperationInstructionServiceImpl.class.getName());
```

在log4jMethodServer.properties增加

``` java
log4j.logger.ext.wbgy.service.impl.OperationInstructionServiceImpl=INFO,methodServerLogFile,myCustomLog
log4j.additivity.ext.wbgy.service.impl.OperationInstructionServiceImpl=false
```
**输出标准**
``` java
logger.info("User {} logged in at {}", username, loginTime); // 规范写法
```
### git
```shell
git fetch -p   从远端仓库获取最新更新 
git stash  若存在当前分支没有写完 先存暂存区  在去切换其他分支 
git checkout feature-1.70
git stash pop      # 恢复暂存的修改
git branch -a  查看所有分支 包括远程分支
git checkout -b  <new-branch > 创建并切换新的分支
```

### 部署手册
1. 编译部署 ant -f wanb.xml jc
2. 配置文件添加datautility类的配置
- 增加DataUtility配置
  wt.services/svc/default/com.ptc.core.components.descriptor.DataUtility/GYZYChangeDataUtility/java.lang.Object/0=ext.wbgy.doc.dataUtility.GYZYChangeDataUtility/duplicate
- 增加filter配置(工艺平台 快速链接 按钮过滤）)
    wt.services/svc/default/com.ptc.core.ui.validation.SimpleValidationFilter/GYLibraryFilter/null/0=ext.wbgy.validator.GYLibraryFilter/duplicate
