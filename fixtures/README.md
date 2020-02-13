sudo chown -R stack:stack ./fixtures

删除证书目录：
rm -Rf crypto-config channel-artifacts
生成证书： 
cryptogen generate --config=crypto-config.yaml

新建目录：
mkdir channel-artifacts
生成创世区块 
configtxgen -profile TwoOrgsOrdererGenesis -outputBlock channel-artifacts/genesis.block

创建通道
echo $CHANNEL_NAME
export CHANNEL_NAME=ipChannel
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx channel-artifacts/channel.tx -channelID $CHANNEL_NAME

生成锚节点
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate channel-artifacts/Org1MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org1MSP

configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate channel-artifacts/Org2MSPanchors.tx -channelID $CHANNEL_NAME -asOrg Org2MSP

启动网络
docker-compose -f docker-compose-cli.yaml up -d

清空网络
docker-compose -f docker-compose-cli.yaml down

进入 Docker cli 容器
docker exec -it cli bash
echo $CHANNEL_NAME
export CHANNEL_NAME=ipChannel
peer channel create -o orderer.paat.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/channel.tx --tls true  --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/paat.com/orderers/orderer.paat.com/msp/tlscacerts/tlsca.paat.com-cert.pem
参数说明:
-o： 指定 orderer 节点的地址
-c： 指定要创建的应用通道的名称(必须与在创建应用通道交易配置文件时的通道名称一致)
-f： 指定创建应用通道时所使用的应用通道交易配置文件
--tls： 开启 TLS 验证
--cafile： 指定 TLS_CA 证书的所在路径

应用通道所包含组织的成员节点可以加入通道中
 peer channel join -b ipChannel.block

join命令： 将本 Peer 节点加入到应用通道中
参数说明：
-b: 指定当前节点要加入/联接至哪个应用通道

更新锚节点
使用 Org1 的管理员身份更新锚节点配置

peer channel update -o orderer.paat.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org1MSPanchors.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/paat.com/orderers/orderer.paat.com/msp/tlscacerts/tlsca.paat.com-cert.pem


使用 Org2 的管理员身份更新锚节点配置
CORE_PEER_LOCALMSPID="Org2MSP"
CORE_PEER_ADDRESS=peer0.org2.paat.com:7051 
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.paat.com/users/Admin@org2.paat.com/msp
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.paat.com/peers/peer0.org2.paat.com/tls/ca.crt 

peer channel update -o orderer.paat.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/Org2MSPanchors.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/paat.com/orderers/orderer.paat.com/msp/tlscacerts/tlsca.paat.com-cert.pem


安装链代码
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
参数说明：
-n： 指定要安装的链码的名称
-v： 指定链码的版本
-p： 指定要安装的链码的所在路径

实例化链码使用 instantiate 命令：
peer chaincode instantiate -o orderer.paat.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/paat.com/orderers/orderer.paat.com/msp/tlscacerts/tlsca.paat.com-cert.pem -C $CHANNEL_NAME -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "OR ('Org1MSP.peer','Org2MSP.peer')"

参数说明:
-o： 指定Oderer服务节点地址
--tls： 开启 TLS 验证
--cafile： 指定 TLS_CA 证书的所在路径
-n： 指定要实例化的链码名称，必须与安装时指定的链码名称相同
-v： 指定要实例化的链码的版本号，必须与安装时指定的链码版本号相同
-C： 指定通道名称
-c： 实例化链码时指定的参数
-P： 指定背书策略
实例化完成后，用户即可向网络中发起交易。

查询链码使用 query 命令实现：
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'

参数说明：
-n： 指定要调用的链码名称
-C： 指定通道名称
-c 指定调用链码时所需要的参数
执行成功输出结果：100

调用链码使用 invoke 命令实现：
peer chaincode invoke -o orderer.paat.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/paat.com/orderers/orderer.paat.com/msp/tlscacerts/tlsca.paat.com-cert.pem  -C $CHANNEL_NAME -n mycc -c '{"Args":["invoke","a","b","10"]}'

参数说明：
-o： 指定orderer节点地址
--tls： 开启TLS验证
--cafile： 指定TLS_CA证书路径
-n: 指定链码名称
-C： 指定通道名称
-c： 指定调用链码的所需参数

执行查询a账户的命令，并查看输出结果：
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'

执行成功输出结果: 90