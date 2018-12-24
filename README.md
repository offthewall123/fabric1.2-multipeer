# fabric1.2-multipeer
fabric1.2多机搭建&amp;通过配置文件加入新组织&amp;通过官网工具动态加入新组织

使用3台服务器节点结构如下
--------
|10.108.233.153|order|
|--|--|
|10.108.233.160| peer0.org1|
|10.108.233.163| peer0.org2|
|10.108.233.153| peer0.org3|

其中peer0.org2通过预先写好配置文件加入通道，peer0.org3通过使用`configtxlator`工具加入通道
## Step0:环境搭建&准备镜像
这一步参照官网教程[Install Samples, Binaries and Docker Images](https://hyperledger-fabric.readthedocs.io/en/release-1.2/install.html),不做赘述，注意一点docker images镜像一定要拉取版本一致的，本次使用的是1.2，镜像版本不一致后期会出现各种各样的问题。
# 正式开始！！！
## Step1:启动order节点，准备生成证书和区块配置文件
