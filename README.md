# Why
Recently, I have been following the [Hyperledger][1] project and [Fabric][4] in particular with fair bit of interest.  The current deployment process[^1] for [Fabric Starter Kit][6] uses [Docker Swarm][2].  

[Kubernetes][5] is a leading platform for automating deployment, scaling, and operations of containerised applications.  It is comparable to [Docker Swarm][2] in terms of target use cases.

Using [Kubernetes][5] would allow [Fabric][4] to leverage features like:
1. Automatic binpacking
2. Horizonta scaling
3. Automated rollouts and rollbacks
4. Storage orchestration
5. Self-healing
6. Service discovery and load balancing
7. Secret and configuration management
8. Batch execution

So this article looks at a simple way to deploy [Fabric Starter Kit][6] on a [Kubernetes][5] platform.  This would be the first step in getting a _production-grade_ [Fabric][4] deployment on [Kubernetes][5].

With developer as the primary audience, we will use [OpenShift][7] as the underlying platform; as [Kubernetes][5] comes bundled in with [OpenShift][7].

# What 
[Fabric][4] application is composed of functional components distributed as linux containers using [Docker][3] packaging format.  

[Kubernetes][5] is an open-source platform for automating deployment, scaling, and operations of application containers across clusters of hosts, providing container-centric infrastructure.

[OpenShift Container Platform - OCP][7] is based on top of [Docker][3] containers and the [Kubernetes][5] container cluster manager.  [OCP][7] adds developer and operational management centric tools to enable rapid application development, easy deployment and scaling, and long-term lifecycle maintenance for small and large teams and applications. 

# How
## Environment
I am using <kbd>macOS Sierra</kbd> for development, with following configuration:
* Mac OSX native [Docker version 1.12.1][9]
* Client tools [version 1.3.1][10] for OpenShift Container Platform

## Steps to deploy [fabric-starter-kit][6] on OpenShift/Kubernetes
1. Bring up OpenShift service
    ```bash
    oc cluster up
    ```
    > First time you execute this command it will take sometime as the service might download required OpenShift images
    
    Note the information within **--Server Information** section  
    >-- Server Information ...   
    OpenShift server started.  
   The server is accessible via web console at:  
       https://192.168.0.7:8443

   >You are logged in as:
       User:     developer
       Password: developer

   >To login as administrator:
       oc login -u system:admin

2. Allow the application to run as <kbd>root</kbd> or within a *privileged* container.
> Allowing application containers to run as <kbd>root</kbd> *by default* should be strongly discouraged[^2]  

   1. Login as an administrator
   ```bash
   oc login -u system:admin
   ```
   2. Allow the *default* user to run applications in a privileged mode (or in [OpenShift][7] speak, within a [privileged security context constraints][8])[^3]
   ```bash
   oc adm policy add-scc-to-user privileged -z default -n myproject
   ```
   3. Logout as administrator and log back in as the default user
   ```bash
   oc login -u developer
   ```
3. Download the docker-compose file for the starter-kit
   ```bash
   wget -O docker-compose-fabric-starter-kit.yml https://raw.githubusercontent.com/hyperledger/fabric/master/examples/sdk/node/docker-compose.yml
   ```
4. Update the downloaded YAML file to ensure appropriate ports are *exposed*[^4].

   ```YAML
   # updated snippet of docker-compose-fabric-starter-kit.yml
   membersrvc:
     ports:
     - "7054:7054"

   peer:
     ports:
     - "7051:7051"
   ```
5. Import the docker-compose application into OpenShift
   ```bash
   oc import docker-compose -f docker-compose-fabric-starter-kit.yml
   ```
6. Allow _member service_ to be executed with _root_ status
   ```bash
   oc patch dc membersrvc -p '{"spec":{"template":{"spec":{"containers":[{"name":"membersrvc","securityContext":{"privileged":true}}]}}}}'
   ```
7. Allow validating peer service _vp0_ to be executed with _root_ status
   ```bash
   oc patch dc vp0 -p '{"spec":{"template":{"spec":{"containers":[{"name":"vp0","securityContext":{"privileged":true}}]}}}}'
   ```
8. Give the system couple of minutes to startup the required containers.  Then ensure that all the services are up and running.
   ```bash
   oc get pods
   ```
   >Result should be similar to below:  
   
   ```bash
   NAME                 READY     STATUS    RESTARTS   AGE  
   membersrvc-2-3ft1y   1/1       Running   0          16m  
   peer-2-3p43z         1/1       Running   0          15m  
   starter-1-h8z1d      1/1       Running   1          16m  
   ```
   
   1. <kbd>membersrvc-2-3ft1y</kbd> is the member service
   2. <kbd>peer-2-3p43z</kbd> is the validating peer service
   3. <kbd>starter-1-h8z1d</kbd> is the fabric-starter-kit service
   
9. Deploy and execute the sample *chaincode[^5]* using Node.js Client SDK within the fabric-starter-kit
   ```bash
   oc exec starter-1-h8z1d node app
   ```
   >Result should be similar to below:  
   ```bash
   *** starting HFC sample ***
   member services address =membersrvc:7054
   peer address =peer:7051
   DEPLOY_MODE=dev
   enrolling user admin ...
   Enrolled JohnDoe successfully
   deploying chaincode; please wait ...
   deploy complete; results: {"uuid":"mycc","chaincodeID":"mycc"}
   invoke chaincode ...
   invoke submitted successfully; results={"uuid":"85782c59-e192-499f-8a5d-108c4423133d"}
   invoke completed successfully; results={"result":"Tx 85782c59-e192-499f-8a5d-108c4423133d complete"}
   querying chaincode ...
   query completed successfully; results={"result":{"type":"Buffer","data":[57,55]}}
   ```
10. Investigate logs for the peer validating service
    ```bash
    oc logs peer-2-3p43z
    ```
    > Result should be similar to:  
    ```bash
    04:06:46.438 [peer] ProcessTransaction -> DEBU 1128 ProcessTransaction processing transaction txid = mycc
    04:06:46.438 [peer] ProcessTransaction -> DEBU 1129 Verifying transaction signature mycc
    04:06:46.438 [crypto] Debugf -> DEBU 112a [validator.vp] Tx confdential level [PUBLIC].
    04:06:46.439 [peer] sendTransactionsToLocalEngine -> DEBU 112b Marshalling transaction CHAINCODE_DEPLOY to send to local engine
    04:06:46.439 [peer] sendTransactionsToLocalEngine -> DEBU 112c Sending message CHAIN_TRANSACTION with timestamp seconds:1477368406 nanos:439541357  to local engine
    04:06:46.439 [consensus/noops] RecvMsg -> DEBU 112d Handling Message of type: CHAIN_TRANSACTION 
    04:06:46.439 [consensus/noops] broadcastConsensusMsg -> DEBU 112e Broadcasting CONSENSUS
    04:06:46.439 [peer] Broadcast -> DEBU 112f Broadcast took 3.096µs
    04:06:46.439 [consensus/noops] RecvMsg -> DEBU 1130 Sending to channel tx uuid: mycc
    04:06:46.559 [peer] ensureConnected -> DEBU 1131 Touch service indicates no dropped connections
    04:06:46.559 [peer] ensureConnected -> DEBU 1132 Connected to: []
    04:06:46.559 [peer] ensureConnected -> DEBU 1133 Discovery knows about: []
    04:06:47.440 [consensus/noops] handleChannels -> DEBU 1134 Process block due to time
    04:06:47.440 [consensus/noops] processTransactions -> DEBU 1135 Starting TX batch with timestamp: seconds:1477368407 nanos:440946960 
    04:06:47.441 [consensus/noops] processTransactions -> DEBU 1136 Executing batch of 1 transactions with timestamp seconds:1477368407 nanos:440946960 
    04:06:47.441 [crypto] Debugf -> DEBU 1137 [validator.vp] Tx confdential level [PUBLIC].
    04:06:47.441 [chaincode] Deploy -> DEBU 1138 user runs chaincode, not deploying chaincode
    04:06:47.442 [state] TxBegin -> DEBU 1139 txBegin() for txId [mycc]
    04:06:47.442 [chaincode] Launch -> DEBU 113a chaincode is running(no need to launch) : mycc
    04:06:47.444 [state] TxFinish -> DEBU 113b txFinish() for txId [mycc], txSuccessful=[true]
    04:06:47.444 [state] GetHash -> DEBU 113c Enter - GetHash()
    04:06:47.445 [buckettree] ComputeCryptoHash -> DEBU 113d Enter - ComputeCryptoHash()
    04:06:47.445 [buckettree] ComputeCryptoHash -> DEBU 113e Returing existing crypto-hash as recomputation not required
    04:06:47.446 [state] GetHash -> DEBU 113f Exit - GetHash()
    04:06:47.447 [consensus/noops] processTransactions -> DEBU 1140 Committing TX batch with timestamp: seconds:1477368407 nanos:440946960 
    04:06:47.447 [state] GetHash -> DEBU 1141 Enter - GetHash()
    04:06:47.448 [buckettree] ComputeCryptoHash -> DEBU 1142 Enter - ComputeCryptoHash()
    04:06:47.448 [buckettree] ComputeCryptoHash -> DEBU 1143 Returing existing crypto-hash as recomputation not required
    04:06:47.448 [state] GetHash -> DEBU 1144 Exit - GetHash()
    04:06:47.449 [indexes] addIndexDataForPersistence -> DEBU 1145 Indexing block number [5] by hash = [6132347bf82b609e6f28c5c8c98627f36cc4b7b64c713bc36af79aafc9ef4d8851ec66c28d7eb6bfec5e04d811e39cd6377f2cda8b2dce55bd6628b41e5fbd61]
    04:06:47.449 [state] AddChangesForPersistence -> DEBU 1146 state.addChangesForPersistence()...start
    04:06:47.449 [state] AddChangesForPersistence -> DEBU 1147 Adding state-delta corresponding to block number[5]
    04:06:47.449 [state] AddChangesForPersistence -> DEBU 1148 Not deleting previous state-delta. Block number [5] is smaller than historyStateDeltaSize [500]
    04:06:47.450 [state] AddChangesForPersistence -> DEBU 1149 state.addChangesForPersistence()...finished
    04:06:47.450 [ledger] resetForNextTxGroup -> DEBU 114a resetting ledger state for next transaction batch
    04:06:47.450 [buckettree] ClearWorkingSet -> DEBU 114b Enter - ClearWorkingSet()
    04:06:47.451 [ledger] CommitTxBatch -> DEBU 114c There were some erroneous transactions. We need to send a 'TX rejected' message here.
    04:06:47.451 [consensus/handler] CommitTxBatch -> DEBU 114d Committed block with 1 transactions, intended to include 1
    04:06:47.451 [consensus/noops] getBlockData -> DEBU 114e Preparing to broadcast with block number 6
    04:06:47.451 [consensus/noops] getBlockData -> DEBU 114f Got the delta state of block number 6
    04:06:47.451 [consensus/noops] notifyBlockAdded -> DEBU 1150 Broadcasting Message_SYNC_BLOCK_ADDED to non-validators
    04:06:47.452 [peer] Broadcast -> DEBU 1151 Broadcast took 3.795µs
    ```

This proves how easy it is to start developing _chaincode applications[^4]_ to run on [Fabric][4] network.

# Next steps
- [ ] Use the fabric-starter-kit and run it as a standalone app in an external OpenShift project
- [ ] Get the *peer* and *member* services to run with non-root privilege

[1]: https://www.hyperledger.org/
[2]: https://www.docker.com/products/docker-swarm
[3]: https://www.docker.com/
[4]: https://hyperledger-fabric.readthedocs.io/en/latest/
[5]: http://kubernetes.io/docs/whatisk8s/
[6]: https://hyperledger-fabric.readthedocs.io/en/latest/starter/fabric-starter-kit/
[7]: https://www.openshift.com/
[8]: https://blog.openshift.com/understanding-service-accounts-sccs/
[9]: https://download.docker.com/mac/stable/Docker.dmg
[10]: https://github.com/openshift/origin/releases/tag/v1.3.1

[^1]: [The current starter-kit uses an application defined within docker-compose.yml](https://raw.githubusercontent.com/hyperledger/fabric/master/examples/sdk/node/docker-compose.yml)  

[^2]: [Architecting containers: First article in a series of four(4)](http://rhelblog.redhat.com/2015/07/29/architecting-containers-part-1-user-space-vs-kernel-space/)  

[^3]: <kbd>myproject</kbd> is the project created by default.  <kbd>oc login -u developer</kbd> command tells you which project you are using.

[^4]: An example of the updated YAML file is (available on github)[https://raw.githubusercontent.com/finiteloopme/hyperledger-on-kubernetes/master/docker-compose-fabric-starter-kit.yml].

[^5]: [Chaincode](https://hyperledger-fabric.readthedocs.io/en/latest/protocol-spec/#213-chaincode-services) is an application-level code stored on the ledger as part of a transaction.

