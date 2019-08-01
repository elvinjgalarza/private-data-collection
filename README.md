## Private-Data Collection Lab
This project demonstrates private-data collection capabilities on Hyperledger Fabric by utilizing the [Marbles](https://github.com/hyperledger/fabric-samples/tree/master/chaincode/marbles02_private) code pattern on the [Bring Your First Network](https://hyperledger-fabric.readthedocs.io/en/release-1.4/build_network.html) tutorial network.

## Disclaimer
Please note: I am not affiliated with Hyperledger Fabric and do not own any of the content displayed here. All modifications to original content are my own. Please review the License header.

## What is Private Data?
[Private data](https://hyperledger-fabric.readthedocs.io/en/release-1.4/private-data/private-data.html) is confidential data that is stored in a private database on each authorized peer, logically separate from the channel ledger data. Access to this data is restricted to one or more organizations on a channel via a private-data collection Configuration definition (**collections_config.json**). Unauthorized organizations will have a hash reference of private data on the channel ledger as evidence of the transaction data. Also, for further privacy, hashes of the private data, not the private data itself, go through the Ordering-Service so this keeps private data confidential from the Orderer.

## What is Private-Data Collection?
[Private-data collection](https://hyperledger-fabric.readthedocs.io/en/release-1.4/private-data/private-data.html) is used to manage confidential data that two or more organizations on a channel want to keep private from other organizations on that channel. The collection definition describes a subset of organizations on a channel entitled to store a set of private data, which by extension implies that only these organizations can transact with the private data.

The peer of an organization has a private database that is logically separate from the channel ledger data. If the unauthorized Orderer wants to interact with its contents, it must use the hash reference of the data that goes through the Ordering-Service.

## Understanding and Other Relevant Questions
**What are some ways to ensure data privacy?**
* Segregate your network into channels
* Restrict data access to certain roles in your organization by building access control into chaincode logic
* Ledger data at rest can be encrypted on the peer, and data in-transit is encrypted via TLS
* Hash or encrypt the data before calling chaincode; if hashing, there must be a means to share the source data; if encrypting, there must be a means to share decryption keys
* Using Fabric, **private-data** keeps ledger data private from other organizations within the channel **(this is what this lab encapsulates)**

**Why is private-data collection important?**
* There might be organizations that have certain sensitive data that they want to keep from other organizations; however, they also wish to interact with those other organizations within the same channel. 

**How is this method different than creating channels – what data is shared and visible?**
* In the context of Private-Data Collection and a channel, Private-Data Collection describes the data that is collected and kept private from other organizations on the same channel. The question refers to two different solutions as mentioned in the first bullet.

**Which organizations can actually “see” the data?**
* The Collection definition (collections_config.json) describes a subset of organizations that are allowed to store a set of private data – de facto means that these organizations are the same ones that can see and/or transact the respective private data

## Tutorial
1. Clone the repository.
```
git clone https://github.com/elvinjgalarza/private-data-collection.git
```
2. Remove any containers and inactive networks that may be up. WARNING! This will remove all stopped containers and inactive networks.
```
docker ps
docker network list
docker system prune
```
3. Go to the where the BYFN script is located.
```
cd private-data-collection/fabric-samples/first-network
```
4. Perform more housekeeping to clean up any previous environments and containers.
```
./byfn.sh down
docker rm -f $(docker ps -a | awk '($2 ~ /dev-peer.*.marblesp.*/) {print $1}')
docker rmi -f $(docker images | awk '($1 ~ /dev-peer.*.marblesp.*/) {print $3}')
```
5. Start up the BYFN network with CouchDB. BYFN establishes a blockchain network with 2 organizations and 2 peers for each organization. CouchDB is used as the database since it allows for rich JSON queries that the Collection file uses.
```
./byfn.sh up -c mychannel -s couchdb
```
6. Allow for an interactive terminal in order to use the [peer API commands](https://hyperledger-fabric.readthedocs.io/en/master/commands/peerchaincode.html?%20chaincode%20instantiate#peer-chaincode-install), effectively becoming the administrator.
```
docker exec -it cli bash
```
7. Install private data configuration and chaincode onto peer0.org1 (by default, step 6 puts us into peer0.org1's perspective, so there is no need to switch to them).
```
peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/
```
8. Do the same for the remaining peers.

Switch to peer1.org1 and install.
```
export CORE_PEER_ADDRESS=peer1.org1.example.com:8051
peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/
```

Let's check how this is happening by looking at the docker containers.
```
docker ps
```
Notice how peer0.org1 routes to port 8051. This is why we did CORE_PEER_ADDRESS = peer1.org1.example.com:8051. 


Switch to org2 by pointing to org2's [membership service provider](https://hyperledger-fabric.readthedocs.io/en/release-1.4/msp.html), and its [root certificate and private-key](https://hyperledger-fabric.readthedocs.io/en/release-1.4/identity/identity.html).
```
export CORE_PEER_LOCALMSPID=Org2MSP
export PEER0_ORG2_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
```
Switch to peer0.org2 and install.
```
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/
```
Switch to peer1.org2 and install.
```
export CORE_PEER_ADDRESS=peer1.org2.example.com:10051
peer chaincode install -n marblesp -v 1.0 -p github.com/chaincode/marbles02_private/go/
```

9. Make the smart contract active on the channel that the peers are on.
```
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile $ORDERER_CA -C mychannel -n marblesp -v 1.0 -c '{"Args":["init"]}' -P "OR('Org1MSP.member','Org2MSP.member')" --collections-config  $GOPATH/src/github.com/chaincode/marbles02_private/collections_config.json
```
10. Store the private data.
```
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export PEER0_ORG1_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```
10. Let's create a Marble object by invoking initMarble(), which calls PutPrivateData() API.
```
export MARBLE=$(echo -n "{\"name\":\"marble1\",\"color\":\"blue\",\"size\":35,\"owner\":\"elvin\",\"price\":99}" | base64 | tr -d \\n)
peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["initMarble"]}'  --transient "{\"marble\":\"$MARBLE\"}"
```
11. Query for the name, color, size, and owner of the private data of marble1 as a peer in org1. As per the Configuration, peerX.org1 is allowed to see this.
```
peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarble","marble1"]}'
```
12. Query for the price of marble1 as a peer in org1. As per the Configuration, peerX.org1 is allowed to see this.
```
peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
```
13. Change to peerX.Org2 and perform Step 11. As per the Configuration, peerX.org2 is allowed to see this.
```
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
export CORE_PEER_LOCALMSPID=Org2MSP
export PEER0_ORG2_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER0_ORG2_CA
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
```
```
peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarble","marble1"]}'
```
14. Perform Step 12. As per the Configuration, peerX.org2 is NOT allowed to see this. 
```
peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
```
Notice how a member that is not authorized by the Configuration is not allowed to view the pricing details of the Marble object.

Private data can also be purged through the blockToLive variable in the Configuration. We performed a transaction and effectively added a block to the chain when we created the Marble object. However, performing queries to the ledger is not a transaction and, therefore, is not a block added to the chain. 

The blockToLive's number indicates how many blocks on the chain must be added before the private data is deleted. This means that we can create or transfer Marble objects , which are satisfactory to be considered transactions, the number of times that is labeled in the blockToLive and expect the private data to be gone. 

**Follow these steps to perform a purge on private data** 

16. Switch back to peer0.org1
```
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export PEER0_ORG1_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```
17. Let's observe the blocks that I was talking about. Keeping the same terminal you're working on open, let's open up another new terminal and perform these commands.
```
docker logs peer0.org1.example.com 2>&1 | grep -i -a -E 'private|pvt|privdata'
```
18. Perform a query in the old terminal. 
```
peer chaincode query -C mychannel -n marblesp -c '{"Args":["readMarblePrivateDetails","marble1"]}'
```
Now perform step 17 again. Notice how no new blocks were added because we performed a query.

19. Create a new Marble object belonging to Elvin. Name is marble2. 
```
export MARBLE=$(echo -n "{\"name\":\"marble2\",\"color\":\"blue\",\"size\":35,\"owner\":\"elvin\",\"price\":99}" | base64 | tr -d \\n)
peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["initMarble"]}' --transient "{\"marble\":\"$MARBLE\"}"
```
Now perform step 17 again. Notice how we added a new block because we performed a method in the chaincode that is satisfactory to be considered a transaction.

Repeat step 18 to query again.

20. Let's transfer marble2 to Joe.
```
export MARBLE_OWNER=$(echo -n "{\"name\":\"marble2\",\"owner\":\"joe\"}" | base64 | tr -d \\n)
peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["transferMarble"]}' --transient "{\"marble_owner\":\"$MARBLE_OWNER\"}"
```

Now perform step 17 again. Notice how we added a new block because we performed a method in the chaincode that is satisfactory to be considered a transaction.

Repeat step 18 to query again.

21. Let's transfer marble2 to Tom.
```
export MARBLE_OWNER=$(echo -n "{\"name\":\"marble2\",\"owner\":\"tom\"}" | base64 | tr -d \\n)
peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["transferMarble"]}' --transient "{\"marble_owner\":\"$MARBLE_OWNER\"}"
```

Now perform step 17 again. Notice how we added a new block because we performed a method in the chaincode that is satisfactory to be considered a transaction.

Repeat step 18 to query again.

22. Let's transfer marble2 to Jerry.
```
export MARBLE_OWNER=$(echo -n "{\"name\":\"marble2\",\"owner\":\"jerry\"}" | base64 | tr -d \\n)
peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n marblesp -c '{"Args":["transferMarble"]}' --transient "{\"marble_owner\":\"$MARBLE_OWNER\"}"
```
Now perform step 17 again. Notice how we added a new block because we performed a method in the chaincode that is satisfactory to be considered a transaction.

Repeat step 18 to query again. 

Notice now that the private data of marble1 is no longer accessible. We've effectively purged the private data.

## License
Hyperledger Project source code files are made available under the Apache License, Version 2.0 (Apache-2.0), located in the [LICENSE](https://github.com/hyperledger/fabric-samples/blob/release-1.4/LICENSE) file in Hyperledger's fabric-samples. Hyperledger Project documentation files are made available under the Creative Commons Attribution 4.0 International License (CC-BY-4.0), available at http://creativecommons.org/licenses/by/4.0/.
