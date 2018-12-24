# Fabric1.2 多机搭建趟坑记录
### 多机通过配置文件加入新组织（org)到channel&使用`configtxlator`工具加入channel。所有用到的配置文件都会一起上传到仓库里做参考。

### 使用3台服务器节点结构如下
|10.108.233.153|order  |
|--|--|
|10.108.233.160| peer0.org1 |
|10.108.233.163| peer0.org2|
|10.108.233.153| peer0.org3|
其中peer0.org2通过预先写好配置文件加入通道，peer0.org3通过使用`configtxlator`工具加入通道
## Step0:环境搭建&准备镜像
这一步参照官网教程[https://hyperledger-fabric.readthedocs.io/en/release-1.2/prereqs.html]，不做赘述，注意一点docker images镜像一定要拉取版本一致的，本次使用的是1.2，镜像版本不一致后期会出现各种各样的问题。


# 正式开始！！！
## Step1:启动order节点，准备生成证书和区块配置文件
`root@153:/home/u1# mkdir multipeer`
`root@153:/home/u1# cd multipeer`
[https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/]下载解压hyperledger-fabric-linux-amd64-1.2.0.tar.gz，scp其中的bin目录至服务器的multipeer/目录下，修改权限
`root@153:/home/u1/multipeer# chmod -R 777 ./bin`

配置好crypto-config.yaml和configtx.yaml文件，上传到multipeer目录下。

**生成公私钥和证书**
`root@153:/home/u1/multipeer# ./bin/cryptogen generate --config=./crypto-config.yaml`

**生成创世区块**
`root@153:/home/u1/multipeer# mkdir channel-artifacts`
`root@153:/home/u1/multipeer# ./bin/configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block`
此时multipeer目录下应该多了两个文件夹，一个是channel-artifacts里面存放了genesis.block。还有一个crypto-config里面存放了order证书一些相关文件和信息，其中tlsca.example.com-cert.pem文件会在之后创建通道和调用链码时用到。

**生成通道配置区块**
` root@153:/home/u1/multipeer# ./bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/mychannel.tx -channelID mychannel`
此时可以看到channel-artifacts目录下多了一个mychannel.tx文件

**转发整个multipeer文件夹到其他两台服务器上**
`root@153:/home/u1# scp -r multipeer u1@10.108.233.160:/home/u1/`
`root@153:/home/u1# scp -r multipeer u1@10.108.233.163:/home/u1/`

**准备order节点的docker配置文件启动容器**
配置docker-compose-orderer.yaml，放到multipeer目录下
`root@153:/home/u1/multipeer# docker-compose -f docker-compose-orderer.yaml up -d`，
启动order节点容器，此时可以docker ps一下应该是有一个容器在运行的。
![https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/order.PNG]
