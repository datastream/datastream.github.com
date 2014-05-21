# 发布平台

**Time-stamp: <2014-05-21 23:16:28 datastream>**

---

* 前言

>写这文章的原因是，我实在看不下去了。

# 正文

在大部分时间，我总觉得发布一个产品到线上不会是大的问题。实际情况是，我看到的产品发布方式都很丑陋。如果要实现灰度发布基本靠人工。

## 设计目标

一个设计良好的发布平台，应该会有一下特性：
1. 支持发布依赖检测
2. 支持发布前后的消息处理
3. 支持灰度发布
4. 支持版本检测

## client端特点

很多人可能会想自己写一个agent实现发布流程。我觉得很多时候不一定合算。

原因如下：

1. 发布agent不一定好搞。java写的资源占用太多，c写的得看程序员的水平。
2. 发布不是每时每刻都在进行的
3. 发布agent的权限问题

实际设计为crontab脚本模式。
例如产品A对应用户deploy。
对deploy用户设置crontab如下：

    */5 * * * * deploy /opt/bin/deploy.sh

    cat /opt/bin/deploy.sh
    #!/bin/bash
    host=`hostname`
    curl -I https://deploy-platform/api/v1/${host} > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        curl https://deploy-platform/api/v1/${host}/deploy|bash
    if

发布用的crontab设置可以依赖puppet设置。

## server端特点

用户交互方式：
1. REST-full API (http basic认证/token/aws sign4签名)
2. web（backbone.js + REST-full API）

后台任务：
1. 通知服务
2. 用户相关任务(各种条件触发任务)

#### API

`GET /api/v1/${host}`返回状态码200，对应需要部署。
`GET /api/v1/${host}/deploy`返回具体的发布脚本。

发布脚本包含以下功能：

1. `per-deploy`用户定义的部署前的运行脚本，包含环境检测，版本比对，版本文件更新等准备工作。如果有任何异常直接将`stderr`的内容推送到发布平台；反之将`stdout`的输出推送到发布平台。
2. `do-deploy`用户定义的部署脚本。包含老版本的退出机制，新版本的启动机制。如果启动有任何异常，直接将`stderr`的内容推送到发布平台；反之将`stdout`的输出推送到发布平台。
3. `post-deploy`用户定义的检测脚本。例如简单的回归测试用例，启动后一段时间内的异常监控。如果检测更新失败，还可以选择版本回滚。

这几个脚本是预定义的模板+用户的设置自动生成的。

`POST /api/v1/project`发布部署请求。

部署请求包含：
1. deb包等环境依赖（optional）
2. 软件版本
3. ci服务器 (optional)
4. 启动关闭脚本
5. 程序单元测试脚本（optional）
6. 部署时间 (optional)
7. 发布服务器列表

`PUT /api/v1/project`更新部署请求。
部署请求包含：
1. deb包等环境依赖（optional）
2. 软件版本（例如ci服务器上的tag）
3. ci服务器 （optional）
4. 启动关闭脚本 (optional)
5. 程序单元测试脚本（optional）
6. 更新时间 (optional)
7. 发布服务器列表 (optional)

`DELETE /api/v1/project/$id`删除部署请求

#### server后台任务
条件触发：
1. 按照用户定义的软件信息，生成发布用的包。
2. 按照用户更新时间，刷新具体的配置
3. 灰度发布的状态处理

通知任务：
1. 部署情况通知
2. 报表汇总


## 实现难点

1. 发布脚本模板的设置。
2. 各种条件触发的逻辑定义。


## More

### ci服务器

这年头ci服务器搭建比较容易了，简单一点的可以直接用docker+现成的开源解决方案。配置写成Dockerfile。

### 打包

打包的方式基本可以用伪代码写成：

mkdir /tmp/deploy;cd /tmp/deploy
if svn {
    svn co ${url}
    find . -name .svn -type d -exec rm -r {}
}
if git {
    svn clone ${url}
    find . -name .git -type d -exec rm -r {}
}
if ftp {
    get ftpfile
}
tar czf ${tag}.tgz .
