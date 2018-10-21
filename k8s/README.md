# Setup fabric environment

### Prerequisite

* Kubernetes cluster
* Client server which can connect kubernetes cluster
* NFS server for MSP storage
* Volumes for peer,orderer,ca,kafka,zookeeper persistent storage
* Client server must setup docker and kubectl


### Generate crypto materials,channel-artifacts,kubernetes yamls

* Mount root dir of NFS server to the path /opt/share of client server

* Download latest [fabric tools](https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/)
  (note:)

 * Replace NFS server url in templates/template_pod_cli.yaml and templates/template_pod_namespace.yaml
 
 * Default network is 1orderer org/3orderers,2peer org/4peers,2cas,3zookeepers/4kafkas
 
 * If create new network

 	```bash
    cd setupCluster
    ./generateALL.sh false "ca:d-ca1*d-ca2;zk_log:d-zklog1,d-zklog2,d-zklog3;zk_data:d-zkdata1,d-zkdata2,d-zkdata3;kafka:d-k1,d-k2,d-k3,d-k4;peer:d-p0,d-p1*d-p2,d-p3;orderer:d-o1,d-o2,d-o3" nfsAddress
    ```

### Start fabric network

```bash
python3 transform/run.py
```

### Check fabric network status

```bash
kubectl get pods --all-namespaces
```

Wait all pods status be Running

### Test inkchain network

* Create channel.tx

```bash
../bin/configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
```

* Create channel update files，update anchor peer of org

```bash
../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Peerorg2Panchor.tx -channelID mychannel -asOrg Peerorg2MSP

../bin/configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Peerorg1MSPanchor.tx -channelID mychannel -asOrg Peerorg1MSP
```

* Copy channel-artifacts to share dir which all orgs will share

```bash
cp -r ./channel-artifacts /opt/share/
```

* Enter cli container in org1

```bash
kubectl exec -it pods name bash -n peerorg1-demo
```

* Create channel

```bash
peer channel create -o orderer0.ordererorg-demo:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile /etc/hyperledger/fabric/orderertls/tlsca.ordererorg-demo-cert.pem
```

* Copy mychannel.block

```bash
cp mychannel.block ./channel-artifacts
```

* Enter cli container of org1，execute join mychannel

```bash
CORE_PEER_ADDRESS=peer0.peerorg1-demo:7051 peer channel join -b ./channel-artifacts/mychannel.block

CORE_PEER_ADDRESS=peer1.peerorg1-demo:7051 peer channel join -b ./channel-artifacts/mychannel.block

```
* Enter cli container of org2，execute join mychannel

```bash
CORE_PEER_ADDRESS=peer0.peerorg2-demo:7051 peer channel join -b ./channel-artifacts/mychannel.block

CORE_PEER_ADDRESS=peer1.peerorg2-demo:7051 peer channel join -b ./channel-artifacts/mychannel.block

```

* Enter cli container of org1，update anchor peer

```bash
peer channel update -o orderer0.ordererorg-demo:7050 -c mychannel -f ./channel-artifacts/Peerorg1MSPanchor.tx --tls true --cafile /etc/hyperledger/fabric/orderertls/tlsca.ordererorg-demo-cert.pem
```

* Enter cli container of org2，update anchor peer

```bash
peer channel update -o orderer0.ordererorg-demo:7050 -c mychannel -f ./channel-artifacts/Peerorg2MSPanchor.tx --tls true --cafile /etc/hyperledger/fabric/orderertls/tlsca.ordererorg-demo-cert.pem
```

* Download chaincode to client machine location /opt/share/channel-artifacts

* Enter cli container of org1, install chaincode
```bash
CORE_PEER_ADDRESS=peer0.peerorg1-demo:7051 peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/peer/channel-artifacts/chaincode_example02

CORE_PEER_ADDRESS=peer1.peerorg1-demo:7051 peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/peer/channel-artifacts/chaincode_example02
```
* Enter cli container of org2, install chaincode
```bash
CORE_PEER_ADDRESS=peer0.peerorg2-demo:7051 peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/peer/channel-artifacts/chaincode_example02

CORE_PEER_ADDRESS=peer1.peerorg2-demo:7051 peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/peer/channel-artifacts/chaincode_example02
```

* Enter org1,instantiate chaincode

```bash
CORE_PEER_ADDRESS=peer0.peerorg1-demo:7051 peer chaincode instantiate -o orderer0.ordererorg-demo:7050 --tls true --cafile /etc/hyperledger/fabric/orderertls/tlsca.ordererorg-demo-cert.pem -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR('Peerorg1MSP.member','Peerorg2MSP.member')"
```

* Enter org1，check invoke and query

```bash
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'

peer chaincode invoke -o orderer0.ordererorg-demo:7050 -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}' --tls true --cafile /etc/hyperledger/fabric/orderertls/tlsca.ordererorg-demo-cert.pem
```

* Delete whole network

```bash
python3 transform/delete.py
```
