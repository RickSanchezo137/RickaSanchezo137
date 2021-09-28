# [返回](/)

# edge impulse

> edge impulse相关

## 基于开发板

### 环境搭建

> 需要将板子与edge impulse平台对接，上传采集的数据

*PS：下面的命令都要在管理员模式下运行*

[Installation (edgeimpulse.com)](https://docs.edgeimpulse.com/docs/cli-installation)

首先要有python3环境和nodejs v 14以上环境

接着官网里写需要`npm install -g edge-impulse-cli --force`，这一步有几个坑，慢慢填

- 首先是npm速度慢的问题，可以添加淘宝源——安装淘宝的cnpm，然后在使用时直接将npm命令替换成cnpm命令即可

```sh
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

- 接着可能会报

![image-20210716151702391](imgs\1.png)

需要安装VS build tool环境

```sh
cnpm install --global --production windows-build-tools
```

这里有俩坑：

1. 停在`Successfully installed Python 2.7`

   [链接](https://blog.csdn.net/oqzuser1234asd/article/details/116169889)

   在`C:\Users\xxx\AppData\Local\Temp`下，创建一个名为`dd_client_.log`的文件；编辑5中创建的文件，加入一行`Closing installer. Return code: 3010.`然后保存

2. 继续报![image-20210716151702391](imgs\1.png)

   解决：[链接](https://github.com/nodejs/node-gyp/issues/2002#issuecomment-587578162)

   ![image-20210716152051247](D:\笔记\秋招\docs\项目\研究相关\imgs\2.png)

   具体位置是在..\node_global，这个可以通过命令设置global位置，网上很多相关资料

   再重新`cnpm install --global --production windows-build-tools`

   > 可能需要再cnpm -g install node-gyp，但我也不知道具体这一步有没有起作用

   最后，将node_global添加进环境变量，运行`edge-impulse-daemon`

   ![image-20210716155303590](imgs\3.png)

   连接成功！

### 第一个demo

#### 数据采集

