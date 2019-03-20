# Introduction
The [Interoperability Working Group](https://wiki.hyperledger.org/display/fabric/Fabric+Interop+Working+Group) of Hyperledger Fabric community has been working on specifying and standardizing an interoperability workflow that allows organizations hosted on different cloud vendors to establish and join business networks independent from the infrastructure where the ordering services are hosted.

The key part of current proposal is consortium management chaincode (cmcc), which is a kind of user chaincode. In this document, we will analyze the limitation of cmcc, and discuss an alternative of implementing it as system chaincode, consortium management system chaincode (cmscc).

# Problems with the "cmcc" solution
1. It is difficult to release and manage cmcc between different organizations.
2. Cmcc needs to be installed and instantiated in different organizations.
3. Cmcc's instantiation policy and endorsement policy are not dynamic enough.
	* The solidification of the instantiation policy makes the management of the entire alliance appear to be somewhat centralized.
	* The endorsement policy has solidified so that the organizations that joined later can't participate in the endorsement, and the endorsement risk is concentrated.
4. Too much work needs to be done by an auxiliary program outside the chain code.
	* compute configupdate.
	* Interpret whether the current proposal has collected enough signatures.
	* Whether the current propoal has expired due to other configupdate commits.

# Advantages of the cmscc(consortium management system chaincode)
1. The chain code is built into the source code and released by the community.
2. System chain code is automatically installed and instantiated
3. The endorsement policy can be implemented in vscc and can be set to SignedByMajorityPeers. Newly joined organizations can automatically participate in endorsements.
4. The chain code can get the latest configblock, and the work that must be done outside of the chain code can now be done in the chain code. We can further solidify many of the logic in the interoperability process into the chain code, which can better standardize the interoperability process.

# POC Scenario
There is currently a channel consisting of two organizations, Org1 and Org2. At almost the same time, Org1 introduced OrgX to join the channel, Org2 introduced OrgY to join the channel.

1. Org1 submits proposal "add_orgx"
2. Org1 signs the proposal "add_orgx" and submits the signature
3. Org2 submits proposal "add_orgy"
4. Org2 signs the proposal "add_orgy" and submits the signature
5. Org2 finds proposal "add_orgx", signs it, and submits the signature
6. The signatures of proposal "add_orgx" is enough, and the cmscc emulation executes configupdate to verify that the proposal is ready to commit.
7. Org2 or Org1 submits config_update_envelope in proposal "add_orgx"
8. The config_update_envelope of "add_orgx" was successfully executed and OrgX was added to the channel.
9. At this point, the state of proposal "add_orgx" becomes "Submitted", and the state of proposal "add_orgy" becomes "Outofdate". Since the config has been upgraded, the version of the "Application" field in the config changes, and the config_update of the proposal "add_orgy" needs to be recalculated.
10. Org2 calls UpdateProposals, the interface will clear the submitted proposal, recalculate and correct config_update
11. Re-collect the signature of proposal "add_orgy" (omitted...)

# Technical details
## What does the cmscc store?
Store all the proposals, each proposal contains the following:
![image](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-interop/raw/master/imgs/proposal_ds.png)

### config is the serialized common.Config structure
![image](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-interop/raw/master/imgs/config_ds.png)

* Where Sequence stores the sequence of the current configblock when calculating ConfigUpdate
* The ChannelGroup stores the user's input. It is a "patch" form that stores the user's original intent. (For example, adding an organization means adding a KV to Application.Groups)
### Config_update is a serialized common.ConfigUpdate structure that stores the ConfigUpdate to be signed
### The value of the signatures field is the serialized common.ConfigSignature structure. Store signatures uploaded by various organizations

## ProposeConfigUpdate process (AddOrgnization, ReconfigOrgnizationMSP is similar to this process)
![image](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-interop/raw/master/imgs/propose_config_update.png)

## GetProposal/UpdateProposals process (ListProposals is similar to this process)
![image](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-interop/raw/master/imgs/get_proposal.png)

## APP design (currently implemented as shell scripts)
After most of the logic is built into the chaincode, the APP design is lighter. The main logic is divided into two parts:
1. Submit requests such as ProposeConfigUpdate, AddSignature, UpdateProposals, etc. according to the user's operation.
2. Listen to the cmscc events and configblock events. After the above event occurs, call the ListProposals interface to get the latest proposal state and display it to the user.

# TODO List
1. Delete Proposal
2. Proposal's access control. Organizations are allowed to update and delete their own proposals.
3. Configurable Endorsement Policy. Endorsement Policy can be configured via the chaincode interface instead of the fixed SignedByMajorityPeers.

# FAQ
Q:After collecting the signatures, why does cmscc not initiate the config update transaction, but need org1 or org2 to initiate?
A:Designing smart contracts should be "passive", and it is a strange behavior for smart contracts to initiate transactions. Technically, by default the peer certificate is not a ChannelWriter, nor is it able to submit transactions to the orderer.
# Demo（Based on fabric-1.4 and fabric-samples-1.4）
Source code：

[fabric-1.4](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-fork/tree/interop-1.4)

[fabric-samples-1.4](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-samples-fork/tree/interop-1.4)

Usage：

* make fabric

```
  cd fabric
  make
```

* Initialize a two-node environment using fabric-samples/first-network

```
  cd fabric-samples/first-network
  ./byfn generate
  ./byfn up
```

* run interop.sh

```
  ./interop.sh
```

