# Hyperledger Fabric network in K8s with External Chaincodes as pods

Refer the tutorial and instructions in this article: 

	https://liujinye.gitbook.io/openshift-docs/hyperledger-fabric/hyperledger-fabric-on-openshift3.11
	https://medium.com/swlh/how-to-implement-hyperledger-fabric-external-chaincodes-within-a-kubernetes-cluster-fd01d7544523

## Install the binaries

```sh
wget https://github.com/hyperledger/fabric/releases/download/v2.0.1/hyperledger-fabric-linux-amd64-2.0.1.tar.gz

tar -xzf hyperledger-fabric-linux-amd64-2.0.1.tar.gz

# Move to the bin path
mv bin/* /bin

# Check that you have successfully installed the tools by executing
configtxgen --version

# Should print the following output:
# configtxgen:
#  Version: 2.0.1
#  Commit SHA: 1cfa5da98
#  Go version: go1.13.4
#  OS/Arch: linux/amd64
```

## Launching the network

```sh
./fabricOps.sh start
```

Create the namespace and launch the workload:

```
kubectl create ns hyperledger
kubectl create -f orderer-service/
kubectl create -f org1/
kubectl create -f org2/
```

## Configure the network

```
peer channel create -o orderer0:7050 -c mychannel -f ./scripts/channel-artifacts/channel.tx --tls true --cafile $ORDERER_CA
peer channel join -b mychannel.block
peer channel fetch 0 mychannel.block -c mychannel -o orderer0:7050 --tls --cafile $ORDERER_CA
```

## Package the chaincode information for the peer and install it

```
cd chaincode/packaging
tar cfz code.tar.gz connection.json
tar cfz marbles-org1.tgz code.tar.gz metadata.json
peer lifecycle chaincode install marbles-org1.tgz
```

## Build and deploy the chaincode

```
docker build -t chaincode/marbles:1.0 .
kubectl create -f chaincode/k8s
```

## Approve the chaincode and commit it to the channel

```
peer lifecycle chaincode approveformyorg --channelID mychannel --name marbles --version 1.0 --init-required --package-id marbles:e001937433673b11673d660d142c722fc372905db87f88d2448eee42c9c63064 --sequence 1 -o orderer0:7050 --tls --cafile $ORDERER_CA --signature-policy "AND ('org1MSP.peer','org2MSP.peer')"

peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name marbles --version 1.0 --init-required --sequence 1 -o -orderer0:7050 --tls --cafile $ORDERER_CA --signature-policy "AND ('org1MSP.peer','org2MSP.peer')"

peer lifecycle chaincode commit -o orderer0:7050 --channelID mychannel --name marbles --version 1.0 --sequence 1 --init-required --tls true --cafile $ORDERER_CA --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt --signature-policy "AND ('org1MSP.peer','org2MSP.peer')"
```

## Invoke and query the chaincode

```
peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["initMarble","marble1","blue","35","tom"]}' --waitForEvent

peer chaincode invoke -o orderer0:7050 --tls true --cafile $ORDERER_CA -C mychannel -n marbles --peerAddresses peer0-org1:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1/peers/peer0-org1/tls/ca.crt --peerAddresses peer0-org2:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2/peers/peer0-org2/tls/ca.crt -c '{"Args":["initMarble","marble2","red","50","tom"]}' --waitForEvent

peer chaincode query -C mychannel -n marbles -c '{"Args":["readMarble","marble1"]}'
```
