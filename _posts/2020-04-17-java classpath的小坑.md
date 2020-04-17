---
title: java classpath 的小坑
layout: post
categories: java classpath 
tags: java classpath
excerpt:  java classpath 
---

### 问题
开发中有个项目需要用脚本定时跑， 下面是脚本的内容;
今天在执行的过程中在大部分电脑上是执行正常的，有一台机器一直报错“找不到或无法加载主类”
```
set application_path=C:\temp\vmsReportEngine_prod\bin
set JAVA_HOME=D:\develop\java\jdk1.8.0_91
cd %application_path%
set classpath = %JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;%JAVA_HOME%\lib;.
java -Djava.ext.dirs="%application_path%\lib;%application_path%" com.ais.action.ReportEngine
```

出错内容
```
C:\temp\vmsReportEngine_prod>run.bat
C:\temp\vmsReportEngine_prod>set application_path=C:\temp\vmsReportEngine_prod\bin
C:\temp\vmsReportEngine_prod>set JAVA_HOME=D:\develop\java\jdk1.8.0_91
C:\temp\vmsReportEngine_prod>cd C:\temp\vmsReportEngine_prod\bin
C:\temp\vmsReportEngine_prod\bin>set classpath = D:\develop\java\jdk1.8.0_91\lib\dt.jar;D:\develop\java\jdk1.8.0_91\lib\tools.jar;D:\develop\java\jdk1.8.0_91\lib;.
C:\temp\vmsReportEngine_prod\bin>java -Djava.ext.dirs="C:\temp\vmsReportEngine_prod\bin\lib" com.ais.action.ReportEngine
错误: 找不到或无法加载主类 com.ais.action.ReportEngine
```

### 解决办法

把配置改成,执行正常，以后要注意 classpath后面不要留空格
```
   set classpath= %JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;%JAVA_HOME%\lib;. 
```

### 后记
这些细节问题，找了好久才解决。希望以后自己多注意些。写代码读取配置时，也注意空格的处理。
遇到问题记录下来，防止以后再出同样的错误。





