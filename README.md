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
***
# 正式开始！！！
## Step1:启动order节点，准备生成证书和区块配置文件
**准备工作**  
`root@153:/home/u1# mkdir multipeer`  
`root@153:/home/u1# cd multipeer`  
[下载解压hyperledger-fabric-linux-amd64-1.2.0.tar.gz](https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/),scp其中的bin目录至服务器的multipeer/目录下，修改权限  
`root@153:/home/u1/multipeer# chmod -R 777 ./bin`  
配置好crypto-config.yaml和configtx.yaml文件，上传到multipeer目录下。  

**生成公私钥和证书**  
`root@153:/home/u1/multipeer# ./bin/cryptogen generate --config=./crypto-config.yaml`  

**生成创世区块**  
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
`root@153:/home/u1/multipeer# docker-compose -f docker-compose-orderer.yaml up -d`  
启动order节点容器，此时可以docker ps一下应该是有一个容器在运行的。  
![orderContainer](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/order.PNG)  

***
## Step2:部署peer0.org1（160机器上)  
**准备docker-compose-peer.yaml**  
上传docker-compose-peer.yaml至multipeer目录下（注意配置文件里ip和组织名需要自己修改)

**准备chaincode**  
上传fudancode02至multeer/chaincode/go/  

**启动第一个peer容器**  
`root@ubuntu16:/home/u1/multipeer# docker-compose -f docker-compose-peer.yaml up -d`  
此时docker ps 一下可以看到机器上有跑两个容器![peerContainer](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/peer0org1.PNG)  

**进入cli容器并且创建channel**  
`root@ubuntu16:/home/u1/multipeer# docker exec -it cli bash`  
配置cafile路径  
`root@3c0fed2a4547:/opt/gopath/src/github.com/hyperledger/fabric/peer# ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem`  
创建channel  
`root@3c0fed2a4547:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/mychannel.tx --tls --cafile $ORDERER_CA`  
这里用到的 --cafile,就是我们之前order那台机器上生成的证书文件  
这个时候我们在cli容器里ls一下会多了一个mychannel.block的通道配置文件  
![mychannelBlock](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/peer0org1mychannel.PNG)  
这个文件很重要，我们需要将其复制出来到宿主机器上，并转发给我们另一台机器。但在此之前先将当前peer加入到这个channel  
`root@3c0fed2a4547:/opt/gopath/src/github.com/hyperledger/fabric/peer#  peer channel join -b mychannel.block`  
![peer0org1JoinSuccess](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/peer0org1JoinSuccess.PNG)

**保存channel配置文件到宿主机器并发送给另一台机器上**  

退出cli容器  
`root@3c0fed2a4547:/opt/gopath/src/github.com/hyperledger/fabric/peer# exit`

复制mychannel.block文件到宿主机器并发送给另一台  
`root@ubuntu16:/home/u1/multipeer# docker cp 3c0fed2a4547:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block ./`  
注意这里docker cp 命令后面跟的id是你起的docker容器id  
发送mychannel.block  
`root@ubuntu16:/home/u1/multipeer# scp mychannel.block u1@10.108.233.163:/home/u1/multipeer/`

**安装链码并执行合约**  
进入容器并安装  
`root@ubuntu16:/home/u1/multipeer# docker exec -it cli bash`  

`root@3c0fed2a4547:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer chaincode install -n mycc -p github.com/hyperledger/fabric/multipeer/chaincode/go/fudancode02 -v 1.0`  
这条命令 -n 代表我们安装这个链码的名字， -p代表链码文件所在的路径，-v 代表链码的版本，这些都可以修改。我们之所以能在容器内访问到链码文件，是因为我们之前的docker-compose-peer.yaml里面有配置容器和宿主机器的磁盘映射。  
安装成功  
![installSuccess](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/peer0org1InstallSuccess.PNG)  

实例化合约代码初始化a为100 b为200
`ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem`  

`peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile $ORDERER_CA -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR ('Org1MSP.peer','Org2MSP.peer')"`

peer上查询a的值为100  
`peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'`  
-C是通道名，-n 合约名, -c query为合约里的函数， 后面的"a"为传参数给他。  
查询成功截图  
![querySuccess](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/querySuccess.PNG)

在peer上进行转账操作  
`peer chaincode invoke --tls --cafile $ORDERER_CA -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'`  
![peer0org1InvokeSuccess](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/peer0org1InvokeSuccess.PNG)
***

## Step3:peer0.org2加入通道  
同样的准备docker配置文件docker-compose-peer.yaml放到multipeer&准备链码上传fudancode02至multeer/chaincode/go/  
启动容器  
`docker-compose -f docker-compose-peer.yaml up -d`  
将mychannel.block 复制到容器的/opt/gopath/src/github.com/hyperledger/fabric/peer 目录下  
`docker cp mychannel.block 07fab6448fe5:/opt/gopath/src/github.com/hyperledger/fabric/peer/`  
docker cp 后面跟的是当前cli容器的ID需要自己修改  

进入容器将peer加入到channel  
` docker exec -it cli bash`  
`peer channel join -b mychannel.block`  
提示Successfully submitted proposal to join channel说明加入成功，此时还需要install一下链码，注意的是一个通道内instantiate过一次链码之后，之后加入通道的组织只需要再install一次即可  
`peer chaincode install -n mycc -p github.com/hyperledger/fabric/multipeer/chaincode/go/fudancode02 -v 1.0`  
query一下  
`peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'`  
![peer0org2querySuccess](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/peer0org2querySuccess.PNG)
因为peer0.org1之前做过一次invoke所以现在query出来a为90没有问题，同一个通道内成员的数据是同步的。  
可以再两台机子上容器内跑命令  
`peer channel getinfo -c mychannel`  
返回的Blockchain info: {"height":3,"currentBlockHash":"rjRFK1bxEXHqBghrSOQU1sjGh4tnp9wGOHQ7G65KdHk=","previousBlockHash":"z5ZuoLBjcpp7CAXhfN/Dv5GMmZcEE4y/1yM6lgU67LM="}  
说明我们第二个org加入channel成功了。  
***  

## 到目前为止，我们通过配置文件加入新的org到channel没有问题，下面介绍一下通过官网提供的`configtxlator`工具加入一个新的org3到channel 
**也可以参照官网的[Adding an Org to a Channel](https://hyperledger-fabric.readthedocs.io/en/release-1.2/channel_update_tutorial.html)这一章节结合着看**


**准备工作**  
在org1机器/home/u1 下新建文件夹multipeerOrg3 复制一份之前bin文件夹到目录下  
准备configtx.yaml和org3-crypto.yaml上传到multipeerOrg3目录下，这两个文件可以从fabric-samples-release1.2->first-network->org3-artifacts目录下找到。  
**生成org3的证书信息**  
`./bin/cryptogen generate --config=./org3-crypto.yaml`  
`mkdir channel-artifacts`  
`export FABRIC_CFG_PATH=$PWD && ./bin/configtxgen -printOrg Org3MSP > ./channel-artifacts/org3.json`  
此时multipeerOrg3目录下多了一个crypto-config目录和channle-artifacts目录  

到order节点的服务器上将multipeer/crypto-config/ordererOrganizations目录复制到mutipeerOrg3/crypto-config/  

进入容器
`docker exec -it cli bash`  
设置变量名  
`export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem  && export CHANNEL_NAME=mychannel`

**获取当前channel配置**  
`peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA`  
可以看到peer目录下多了一个config.pb文件，这个是当前网络的配置信息，将其复制到宿主机器上  
`exit`  
`docker cp d76a5c35c049:/opt/gopath/src/github.com/hyperledger/fabric/peer/config_block.pb ./`  
**使用configtxlator将config.pb转换成可读的json文件**  
`./bin/configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json`  
此时目录下多了一个config.json文件  

**将org3的配置信息加到config.json中，输出新文件为modified_config.json**  
`jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./channel-artifacts/org3.json > modified_config.json`  
此时目录下多了一个modified_config.json，config.json是最初包含org1和org2的信息，modified_config.json包含了三个org的信息。  

**将config.json转换为config.pb**  
`./bin/configtxlator proto_encode --input config.json --type common.Config --output config.pb`  
此时目录下多了一个config.pb文件  

**将modified_config.json转换为modified_config.pb**    
`./bin/configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb`  
此时目录下多了modified_config.pb文件  

**计算modified_config.json.pb和config.pb的差异输出到org3_update.pb**  
`./bin/configtxlator compute_update --channel_id mychannel --original config.pb --updated modified_config.pb --output org3_update.pb`  
此时目录下多了一个org3_update.pb  

**将org3_update.pb转换成org3_update.json**  
`./bin/configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate | jq . > org3_update.json`  
此时目录下多了一个org3_update.json  

**将json提案文件转为二进制提案文件**  
`echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json`  
此时目录下多了org3_update_in_envelope.json  
`./bin/configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb`  
此时目录下多了org3_update_in_envelope.pb这个文件是我们最终需要用来更新通道信息的文件。  


**签名并提交更新请求**  
将上一步得到的org3_update_in_envelope.pb复制到cli容器内  
`docker cp org3_update_in_envelope.pb d76a5c35c049:/opt/gopath/src/github.com/hyperledger/fabric/peer/`  
`docker exec -it cli bash`  
签名  
`peer channel signconfigtx -f org3_update_in_envelope.pb`  
![peer0org1signSuccess](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/peer0org1SignSuccess.PNG)

**导出org2环境变量**  
`export CORE_PEER_LOCALMSPID="Org2MSP"`  
`export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt`  
`export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp`  
`export CORE_PEER_ADDRESS=peer0.org2.example.com:7051`  

**发送更新请求**  
`peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA`  
![sendUpdateSuccess](https://github.com/offthewall123/fabric1.2-multipeer/blob/master/imgs/sendUpdateSuccess.PNG)  


**编写org3的yaml文件docker-compose-org3.yaml**  
extra_hosts 里需要配置好order节点的ip和当前两个org锚节点的ip，和自己的ip。上传到multipeerOrg3目录下。  
将整个multipeerOrg3文件夹发到到order节点的服务器上。  
`scp -r multipeerOrg3/ u1@10.108.233.153:/home/u1/`  

**启动org3容器加入channel**  
`docker-compose -f docker-compose-org3.yaml up -d`  
`docker exec -it Org3cli bash`  
`export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem && export CHANNEL_NAME=mychannel`  

使用peer channel fetch 获取mychannel.block  
`peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA`  
可以看到peer目录下多了一个mychannel.block文件，有了这个文件之后就可以加入到通道了  
`peer channel join -b mychannel.block`  
我们执行一下`peer channel getinfo -c mychannel` 得到  
`Blockchain info: {"height":4,"currentBlockHash":"SMSXuf+KMk5a/0OHdoLbnkJuQzQ5iZDOIdcqovMKTtA=","previousBlockHash":"rjRFK1bxEXHqBghrSOQU1sjGh4tnp9wGOHQ7G65KdHk="}`  
在其他两台机器上输入同样命令得到了同样的Blockchain info，说明到这步为止，我们org3加入到这个channel成功了，还差最后一步链码的一些操作。

**升级更新链码**  
在org3 cli中执行  
`peer chaincode install -n mycc -v 2.0 -p github.com/hyperledger/fabric/multipeer/chaincode/go/fudancode02/`  
注意这里的-v升级成了2.0  
