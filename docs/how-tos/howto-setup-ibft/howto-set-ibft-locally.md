---
id: howto-set-ibft-locally
title: How to set up IBFT locally
---

:::caution This guide is for testing purposes only

The below guide will instruct you how to set up an IBFT network on your local machine for testing and development
purposes.

The procedure differs greatly from the way you would want to setup the IBFT network for a real use scenario on
a cloud provider: [How to set IBFT on the cloud](/docs/how-tos/howto-setup-ibft/howto-set-ibft-on-the-cloud)

:::


## Requirements

### Use the develop branch

The main Polygon SDK version is located on the [develop branch](https://github.com/0xPolygon/polygon-sdk/tree/develop), and is considered to be a stable version of the SDK,
while other branches are mid-feature implementations.

As the develop branch is the default one, simply running:

```
git clone https://github.com/0xPolygon/polygon-sdk.git
```

will fetch the latest stable source code.

### Go version

The required version of the Go programming language is `>=1.16`.

## Overview

![Local setup](/img/ibft-setup/local.svg)

In this guide our goal is to establish a working `polygon-sdk` blockchain network working with [IBFT consensus protocol](https://github.com/ethereum/EIPs/issues/650).
The blockchain network will consist of 4 nodes of whom all 4 are validator nodes, and as such are eligible for both proposing block, and validating blocks that came from other proposers.
All 4 nodes will run on the same machine, as the idea of this guide is to give you a fully functional IBFT cluster in the least amount of time.

In order to achieve that, we will guide you through 4 easy steps:

1. Initializing data directories will generate both the validator keys for each of the 4 nodes, and initialize empty blockchain data directories. The validator keys are important as we need to bootstrap the genesis block with the initial set of validators using these keys.
2. Preparing the connection string for the bootnode will be the vital information for every node we will run as to which node to connect to when starting for the first time.
3. Generating the `genesis.json` file will require as input both the validator keys generated in **step 1** used for setting the initial validators of the network in the genesis block, and the bootnode connection string from **step 2**.
4. Running all the nodes is the end goal of this guide and will be the last step we do, we will instruct the nodes which data directory to use and where to find the `genesis.json` which bootstraps the initial network state.

As all four nodes will be running on localhost, during the setup process it is expected that all the data directories
for each of the nodes are in the same parent directory.

## Step 1: Initialize data folders for IBFT and generate validator keys

In order to get up and running with IBFT, you need to initialize the data folders,
one for each node:

````bash
go run main.go ibft init --data-dir test-chain-1
````

````bash
go run main.go ibft init --data-dir test-chain-2
````

````bash
go run main.go ibft init --data-dir test-chain-3
````

````bash
go run main.go ibft init --data-dir test-chain-4
````

Each of these commands will print the validator key and the [node ID](https://docs.libp2p.io/concepts/peer-id/). You will need the Node ID of the first node for the next step.

## Step 2: Prepare the multiaddr connection string for the bootnode

For a node to successfully establish connectivity, it must know which `bootnode` server to connect to in order gain
information about all the remaining nodes on the network. The `bootnode` is sometimes also known as the `rendezvous` server in p2p jargon.

`bootnode` is not  a special instance of the polygon-sdk node. Every polygon-sdk node can serve as a `bootnode`, but
every polygon-sdk node needs to have a set of bootnodes specified which will be contacted to provide information on how to connect with
all remaining nodes in the network.

In order to create the connection string for specifying the bootnode, we will need to conform 
to the [multiaddr format](https://docs.libp2p.io/concepts/addressing/):
```
/ip4/<ip_address>/tcp/<port>/p2p/<node_id>
```

In this guide, we will treat the first node as the bootnode for all other nodes. What will happen in this scenario
is that `nodes 2-4` connecting to the `node 1` will get information on how to connect to one another through the mutually
contacted `node 1`.

Since we are running on localhost, it is safe to assume that the `<ip_address>` is `127.0.0.1`.

For the `<port>` we will use `10001` since we will configure the libp2p server for `node 1` to listen on this port later.

And lastly, we need the `<node_id>` which we can get from the output of the previously ran command `go run main.go ibft init --data-dir test-chain-1` command (which was used to generate keys and data directories for the `node1`)

After the assembly, the multiaddr connection string to the `node 1` which we will use as the bootnode will look something like this (only the `<node_id>` which is at the end should be different):
```
/ip4/127.0.0.1/tcp/10001/p2p/16Uiu2HAmJxxH1tScDX2rLGSU9exnuvZKNM9SoK3v315azp68DLPW
```

## Step 3: Generate an IBFT genesis file with the 4 nodes as validators

````bash
go run main.go genesis --consensus ibft --ibft-validators-prefix-path test-chain- --bootnode /ip4/127.0.0.1/tcp/10001/p2p/16Uiu2HAmJxxH1tScDX2rLGSU9exnuvZKNM9SoK3v315azp68DLPW
````

What this command does:

* The `--ibft-validators-prefix-path` sets the prefix folder path to the one specified which IBFT in Polygon SDK can
  use. This directory is used to house the `consensus/` folder, where the validator's private key is kept. The
  validator's public key is needed in order to build the genesis file - the initial list of bootstrap nodes.
  This flag only makes sense when setting up the network on localhost, as in a real world scenario we cannot expect all
  the nodes' data directories to be on the same filesystem from where we can easily read their public keys.
* The `--bootnode` sets the address of the bootnode that will enable the nodes to find each other.
  We will use the multiaddr string of the `node 1`, as mentioned in **step 2**.

The result of this command is the `genesis.json` file which contains the genesis block of our new blockchain, with the predefined validator set and the configuration for which node to contact first in order to establish connectivity.

:::info Premining account balances

You will probably want to set up your blockchain network with some addresses having "premined" balances.

To achieve this, pass as many `--premine` flags as you want per address that you want to be initialized with a certain balance
on the blockchain.

Example if we would like to premine 1000 ETH to address `0x3956E90e632AEbBF34DEB49b71c28A83Bc029862` in our genesis block, then we would need to supply the following argument:

```
--premine=0x3956E90e632AEbBF34DEB49b71c28A83Bc029862:1000000000000000000000
```

**Note that the premined amount is in WEI, not ETH.**

:::


## Step 4: Run all the clients

Because we are attempting to run the IBFT network consisting of 4 nodes all on the same machine, we need to take care to 
avoid port conflicts. This is why we will use the following reasoning for determining the listening ports of each server of a node:

- `10000` for the gRPC server of `node 1`, `20000` for the GRPC server of `node 2`, etc.
- `10001` for the libp2p server of `node 1`, `20001` for the libp2p server of `node 2`, etc.
- `10002` for the JSON-RPC server of `node 1`, `20002` for the JSON-RPC server of `node 2`, etc.

To run the **first** client (note the port `10001` since it was used as a part of the libp2p multiaddr in **step 2** alongside with node 1's Node ID):

````bash
go run main.go server --data-dir ./test-chain-1 --chain genesis.json --grpc :10000 --libp2p :10001 --jsonrpc :10002 --seal
````

To run the **second** client:

````bash
go run main.go server --data-dir ./test-chain-2 --chain genesis.json --grpc :20000 --libp2p :20001 --jsonrpc :20002 --seal
````

To run the **third** client:

````bash
go run main.go server --data-dir ./test-chain-3 --chain genesis.json --grpc :30000 --libp2p :30001 --jsonrpc :30002 --seal
````

To run the **fourth** client:

````bash
go run main.go server --data-dir ./test-chain-4 --chain genesis.json --grpc :40000 --libp2p :40001 --jsonrpc :40002 --seal
````

To briefly go over what has been done so far:

* The directory for the client data has been specified to be **./test-chain-\***
* The GRPC servers have been started on ports **10000**, **20000**, **30000** and **40000**, for each node respectively
* The libp2p servers have been started on ports **10001**, **20001**, **30001** and **40001**, for each node respectively
* The JSON-RPC servers have been started on ports **10002**, **20002**, **30002** and **40002**, for each node respectively
* The *seal* flag means that the node being started is going to participate in block sealing
* The *chain* flag specifies which genesis file should be used for chain configuration

The structure of the genesis file is covered in the [CLI Commands](/docs/cli-commands) section.

After running the previous commands, you have set up a 4 client IBFT network, capable of sealing blocks and recovering
from node failure.

## Step 5: Interact with the polygon-sdk network

Now that you've set up at least 1 running client, you can go ahead and interact with the blockchain using the account you premined above
and by specifying the JSON-RPC URL to any of the 4 nodes:
- Node 1: `http://localhost:10002`
- Node 2: `http://localhost:20002`
- Node 3: `http://localhost:30002`
- Node 4: `http://localhost:40002`

Follow this guide to issue operator commands to the newly built cluster: [How to query operator information](/docs/how-tos/howto-query-operator) (the GRPC ports for the cluster we have built are `10000`/`20000`/`30000`/`40000` for each node respectively)