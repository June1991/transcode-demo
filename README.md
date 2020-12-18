## 应用简介
使用COS+云函数+CLS+FFmpeg，快速构建高可用、并行处理、实时日志、高度自定义视频转码服务。

应用优势：

- 流式转码，流式读写cos，转码大小不受云函数内存大小限制。
- 实时日志输出，可实时查看转码进度。
- 函数长时运行，最高能完成12小时内的转码应用服务。

应用创建的资源：

- 云函数 ： 流式读取cos文件，使用ffmpeg转码后流式输出回cos中，并将转码过程的实时日志输出到cls。
- 层 ： 转码需要使用的云函数runtime下的ffmpeg可执行文件。
- cls日志 ：存储转码过程的实时日志。cls日志可能会产生一定计费，详情参考[cls计费规则](https://cloud.tencent.com/document/product/614/47116)。

注意事项：

1.  转码应用需要依赖云函数长时运行能力，目前该能力还在公测中，需要先申请白名单。
2.  转码输出桶与函数建议配置在同一区域，因为跨区域配置转码应用稳定性及效率都会降低，并且会产生跨区流量费用。
3.  需要为转码应用的云函数创建运行角色，并授权cos读写权限。详细配置参考[运行角色](#运行角色)。
4.   FFmpeg转码工具对于不同格式之间的转码，参数会有区别，本文仅提供几个样例作为转码参考，更多转码参数使用请参考[FFmpeg官网](https://ffmpeg.org/documentation.html)

## 前提条件

1. [安装serverlesss framework](https://cloud.tencent.com/document/product/1154/42990)
2. 应用依赖函数长时运行能力，目前该能力还在公测中，需要先申请白名单。
3. 配置部署账号权限。参考 [账号和权限配置](https://cloud.tencent.com/document/product/1154/43006) 
4. 配置[运行角色](#运行角色)权限。

## 操作步骤
1. 下载[转码应用](https://cloud.tencent.com/document/product/1154/42990)。

2. 解压文件，进入项目目录transcode-demo，将看到目录结构如下：

   ```
   transcode-demo
   |- .env  #环境配置
   |- serverless.yml # 应用配置
   |- log/ #log日志配置
   |  └── serverless.yml
   └──transcode/  #转码函数配置
      |- ffmpeg   #转码ffmpeg工具
      |- index.py
      |- task_report.py
      └── serverless.yml
   
   ```

   a)  log/serverless.yml 定义一个cls日志集和主题，用于转码过程输出的日志保存，目前采用腾讯云cls日志存储。每个转码应用将会根据配置的cls日志集和主题去创建相关资源，cls的使用会产生计费，具体参考[cls计费规则](https://cloud.tencent.com/document/product/614/47116)。

   b)  transcode/serverless.yml 定义函数的基础配置及转码参数配置。

   c)  transcode/index.py 转码功能实现。

   d)  transcode/task_report.py cls日志上报接口。

   e)   transcode/ffmpeg 转码工具ffmpeg。

3. 配置环境变量和应用参数

   a)  应用参数，文件transcode-demo/serverless.yml

   ```
   #应用信息
   app: video-demo # 您需要配置成您的应用名称
   stage: dev # 环境名称，默认为dev
   ```

   a)  环境变量，文件transcode-demo/.env

   ```
   REGION=ap-shanghai  # 应用创建所在区，目前只支持上海区
   TENCENT_SECRET_ID=xxxxxxxxxxxx # 您的腾讯云sercretId
   TENCENT_SECRET_KEY=xxxxxxxxxxxx # 您的腾讯云sercretKey
   ```

   >?
   >
   >- 登陆腾讯云账号，可以在 [API 密钥管理](https://console.cloud.tencent.com/cam/capi) 中获取 SecretId 和 SecretKey。
   >- 如果您的账号为主账号，或者子账号具有扫码权限，也可以不配置sercretId与sercretKey，直接扫码部署应用。更多详情参考[账号和权限配置](https://cloud.tencent.com/document/product/1154/43006)。

4. 配置转码需要的参数信息

   a)  cls日志定义，文件transcode-demo/log/serverless.yml

   ```
   #组件信息
   component: cls # (必填) 引用 component 的名称
   name: cls-video # (必填) 创建的实例名称，请修改成您的实例名称
   
   #组件参数
   inputs:
     name: cls-log  # 您需要配置一个name，作为您的cls日志集名称
     topic: video-log # 您需要配置一个topic，作为您的cls日志主题名称
     region: ${env:REGION} # 区域，统一在环境变量中定义
     period: 7 # 日志保存时间，单位天
   ```

   b)  云函数及转码配置，文件transcode-demo/transcode/serverless.yml ：  

   ```
   #组件信息
   component: scf # (必填) 引用 component 的名称
   name: transcode-video # (必填) 创建的实例名称，请修改成您的实例名称
   
   #组件参数
   inputs:
     name: transcode-video-${app}-${stage}
     src: ./src
     handler: index.main_handler 
     role: transcodeRole # 函数执行角色，已授予cos对应桶全读写权限
     runtime: Python3.6 
     memorySize: 3072 # 内存大小，单位MB
  timeout: 43200 # 函数执行超时时间，单位秒
     region: ${env:REGION} # 函数区域，统一在环境变量中定义，
  asyncRunEnable: true # 是否支持长时运行，目前只支持上海区
     cls: # 函数日志
       logsetId: ${output:${stage}:${app}:cls-video.logsetId}  # cls日志集
       topicId: ${output:${stage}:${app}:cls-video.topicId}  # cls日志主题
     layers: # layer配置
       - name: ${output:${stage}:${app}:ffmpeg-layer.name} # layer名称
         version: ${output:${stage}:${app}:ffmpeg-layer.version} # 版本
     environment: 
       variables:  # 转码参数
         REGION: ${env:REGION} # 输出桶区域
         DST_BUCKET: test-123456789 # 输出桶名称
         DST_PATH: outputs/ # 输出桶路径
         DST_FORMATS: avi # 转码生成格式
         DST_CONFIG: ffmpeg -i {inputs} -y -f {outputs}  # 转码命令
         TZ: Aisa/Shanghai # cls日志输出时间的时区
     events:
       - cos: # cos触发器    	
           parameters:          
             bucket: test-123456789.cos.ap-shanghai.myqcloud.com  # 输入文件桶
             filter:
               prefix: video/inputs/  # 桶内路径
             events: 'cos:ObjectCreated:*'  # 触发事件
             enable: true
   ```
   
   >?
   >
   >- 输出桶与函数建议配置在同一区域，跨区域配置应用稳定性及效率都会降低，并且会产生跨区流量费用。
   >- 内存大小上限为3072 MB，运行时长上限为43200 s。如需调整，请提工单申请配额调整。
   >- 转码应用必须开启函数长时运行asyncRunEnable: true。
   >- 运行角色请根据[运行角色](#运行角色)创建并授权。
   
5. 在`transcode-demo`项目目录下，执行`sls deploy`部署项目。

   ```
   cd transcode-demo && sls deploy
   ```

6. 上传视频文件到已经配置好的cos桶指定路径，则会自动转码。本示例中是cos桶test-123456789.cos.ap-shanghai.myqcloud.com下的/video/inputs/

7.  转码成功后，文件将保存在您配置的输出桶路径中。本示例中是cos桶test-123456789.cos.ap-shanghai.myqcloud.com下的/video/outputs/

8. 如果需要调整转码配置，修改文件transcode/serverless.yml 后，重新部署云函数即可：

   ```
   cd transcode && sls deploy
   ```

##  监控与日志  

批量文件上传到cos会并行触发转码执行。

1. 登陆云函数控制台，查看日志监控。
2. 直接点击函数对应的cls日志，查看日志检索分析 。

## 自定义FFmpeg

转码应用场景中默认使用的是ffmpeg v1.1版本，如果您想自定义ffmpeg版本，执行以下操作：

1. 将样例中的ffmpeg替换成你自定义的ffmpeg版本。
2. transcode-demo项目目录下再次执行sls deploy部署更新。

```
 cd transcode-demo && sls deploy
```

## 运行角色

转码函数运行时需要读取cos资源进行转码，并将转码后的资源写回cos，因此需要给函数配置一个授权cos全读写的运行角色。更多参考[函数运行角色](https://cloud.tencent.com/document/product/583/47933#.E8.BF.90.E8.A1.8C.E8.A7.92.E8.89.B2)。

1. 登录 [访问管理](https://console.cloud.tencent.com/cam/role) 控制台，选择新建角色，角色载体为腾讯云产品服务。

   

2. 在“输入角色载体信息”步骤中勾选【云函数（scf）】，并单击【下一步】：

3. 在“配置角色策略”步骤中，选择函数所需策略并单击【下一步】。如下图所示：

   

   > 说明：
   >
   > 您可以直接选择 `QcloudCOSFullAccess` 对象存储（COS）全读写访问权限，如果需要更细粒度的权限配置，请根据实际情况配置选择。

4. 输入角色名称，完成创建角色及授权。