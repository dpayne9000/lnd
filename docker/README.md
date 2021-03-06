This document is written for people who are eager to do something with 
the Lightning Network Daemon (`lnd`). This folder uses `docker-compose` to
package `lnd` and `btcd` together to make deploying the two daemons as easy as
typing a few commands. All configuration between `lnd` and `btcd` are handled
automatically by their `docker-compose` config file.

### Prerequisites
    Name  | Version 
    --------|---------
    docker-compose | 1.9.0
    docker | 1.13.0
  
### Table of content
 * [Create lightning network cluster](#create-lightning-network-cluster)
 * [Connect to faucet lightning node](#connect-to-faucet-lightning-node)
 * [Questions](#questions)

### Create lightning network cluster
This section describes a workflow on `simnet`, a development/test network
that's similar to Bitcoin Core's `regtest` mode. In `simnet` mode blocks can be
generated at will, as the difficulty is very low. This makes it an ideal
environment for testing as one doesn't need to wait tens of minutes for blocks
to arrive in order to test channel related functionality. Additionally, it's
possible to spin up an arbitrary number of `lnd` instances within containers to
create a mini development cluster. All state is saved between instances using a
shared value.

Current workflow is big because we recreate the whole network by ourselves,
next versions will use the started `btcd` bitcoin node in `testnet` and
`faucet` wallet from which you will get the bitcoins.

In the workflow below, we describe the steps required to recreate following
topology, and send a payment from `Alice` to `Bob`.
```
+ ----- +                   + --- +
| Alice | <--- channel ---> | Bob |  <---   Bob and Alice are the lightning network daemons which 
+ ----- +                   + --- +         creates the channel and interact with each other using   
    |                          |            Bitcoin network as source of truth. 
    |                          |            
    + - - - -  - + - - - - - - +            
                 |
        + --------------- +
        | Bitcoin network |  <---  In current scenario for simplicity we create only one  
        + --------------- +        "btcd" node which represents the Bitcoin network, in  
                                    real situation Alice and Bob will likely be 
                                    connected to different Bitcoin nodes.
```

**General workflow is the following:** 

 * Create a `btcd` node running on a private `simnet`.
 * Create `Alice`, one of the `lnd` nodes in our simulation network.
 * Create `Bob`, the other `lnd` node in our simulation network.
 * Mine some blocks to send `Alice` some bitcoins.
 * Open channel between `Alice` and `Bob`.
 * Send payment from `Alice` to `Bob`.
 * Close the channel between `Alice` and `Bob`.
 * Check that on-chain `Bob` balance was changed.

Start `btcd`, and then create an address for `Alice` that we'll directly mine
bitcoin into.
```bash
# Init bitcoin network env variable:
$ export NETWORK="simnet" 

# Run the "Alice" container and log into it:
$ docker-compose run -d --name alice lnd_btc
$ docker exec -i -t alice bash

# Generate a new backward compatible nested p2sh address for Alice:
alice$ lncli newaddress np2wkh 

# Recreate "btcd" node and set Alice's address as mining address:
$ MINING_ADDRESS=<alice_address> docker-compose up -d btcd

# Generate 400 block (we need at least "100 >=" blocks because of coinbase 
# block maturity and "300 ~=" in order to activate segwit):
$ docker-compose run btcctl generate 400

# Check that segwit is active:
$ docker-compose run btcctl getblockchaininfo | grep -A 1 segwit
```

Check `Alice` balance:
```
alice$ lncli walletbalance --witness_only=true
```

Connect `Bob` node to `Alice` node.

```bash
# Run "Bob" node and log into it:
$ docker-compose run -d --name bob lnd_btc
$ docker exec -i -t bob bash

# Get the identity pubkey of "Bob" node:
bob$ lncli getinfo

{
    ----->"identity_pubkey": "0343bc80b914aebf8e50eb0b8e445fc79b9e6e8e5e018fa8c5f85c7d429c117b38",
    "alias": "",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 1215,
    "block_hash": "7d0bc86ea4151ed3b5be908ea883d2ac3073263537bcf8ca2dca4bec22e79d50",
    "synced_to_chain": true,
    "testnet": false
    "chains": [
        "bitcoin"
    ]
}

# Get the IP address of "Bob" node:
$ docker inspect bob | grep IPAddress

# Connect "Alice" to the "Bob" node:
alice$ lncli connect <bob_pubkey>@<bob_host>

# Check list of peers on "Alice" side:
alice$ lncli listpeers
{
    "peers": [
        {
            "pub_key": "0343bc80b914aebf8e50eb0b8e445fc79b9e6e8e5e018fa8c5f85c7d429c117b38",
            "peer_id": 1,
            "address": "172.19.0.4:9735",
            "bytes_sent": "357",
            "bytes_recv": "357",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": true,
            "ping_time": "0"
        }
    ]
}

# Check list of peers on "Bob" side:
bob$ lncli listpeers
{
    "peers": [
        {
            "pub_key": "03d0cd35b761f789983f3cfe82c68170cd1c3266b39220c24f7dd72ef4be0883eb",
            "peer_id": 1,
            "address": "172.19.0.3:51932",
            "bytes_sent": "357",
            "bytes_recv": "357",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": false,
            "ping_time": "0"
        }
    ]
}
```

Create the `Alice<->Bob` channel.
```bash
# Open the channel with "Bob":
alice$ lncli openchannel --node_key=<bob_identity_pubkey> --local_amt=1000000

# Include funding transaction in block thereby open the channel:
$ docker-compose run btcctl generate 1

# Check that channel with "Bob" was created:
alice$ lncli listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "0343bc80b914aebf8e50eb0b8e445fc79b9e6e8e5e018fa8c5f85c7d429c117b38",
            "channel_point": "3511ae8a52c97d957eaf65f828504e68d0991f0276adff94c6ba91c7f6cd4275:0",
            "chan_id": "1337006139441152",
            "capacity": "1005000",
            "local_balance": "1000000",
            "remote_balance": "0",
            "unsettled_balance": "0",
            "total_satoshis_sent": "0",
            "total_satoshis_received": "0",
            "num_updates": "0"
        }
    ]
}
```

Send the payment form `Alice` to `Bob`.
```bash
# Add invoice on "Bob" side:
bob$ lncli addinvoice --value=10000
{
        "r_hash": "<your_random_rhash_here>", 
        "pay_req": "<encoded_invoice>", 
}

# Send payment from "Alice" to "Bob":
alice$ lncli sendpayment --pay_req=<encoded_invoice>

# Check "Alice"'s channel balance
alice$ lncli channelbalance

# Check "Bob"'s channel balance
bob$ lncli channelbalance
```

Now we have open channel in which we sent only one payment, let's imagine
that we sent lots of them and we'll now like to close the channel. Let's do
it!
```bash
# List the "Alice" channel and retrieve "channel_point" which represent
# the opened channel:
alice$ lncli listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "0343bc80b914aebf8e50eb0b8e445fc79b9e6e8e5e018fa8c5f85c7d429c117b38",
       ---->"channel_point": "3511ae8a52c97d957eaf65f828504e68d0991f0276adff94c6ba91c7f6cd4275:0",
            "chan_id": "1337006139441152",
            "capacity": "1005000",
            "local_balance": "990000",
            "remote_balance": "10000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "10000",
            "total_satoshis_received": "0",
            "num_updates": "2"
        }
    ]
}

# Channel point consist of two numbers separated by colon the first one 
# is "funding_txid" and the second one is "output_index":
alice$ lncli closechannel --funding_txid=<funding_txid> --output_index=<output_index>

# Include close transaction in block thereby close the channel:
$ docker-compose run btcctl generate 1

# Check "Alice" on-chain balance was credited by her settled amount in the channel:
alice$ lncli walletbalance

# Check "Bob" on-chain balance was credited with the funds he received in the
# channel:
bob$ lncli walletbalance
{
    "balance": 0.0001
}
```

### Connect to faucet lightning node
In order to be more confident with `lnd` commands I suggest you to try 
to create a mini lightning network cluster ([Create lightning network cluster](#create-lightning-network-cluster)).

In this section we will try to connect our node to the faucet/hub node 
which will create with as the channel and send as some amount of 
bitcoins. The schema will be following:

```
+ ----- +                   + ------ +         (1)        + --- +
| Alice | <--- channel ---> | Faucet |  <--- channel ---> | Bob |    
+ ----- +                   + ------ +                    + --- +        
    |                            |                           |           
    |                            |                           |      <---  (2)         
    + - - - -  - - - - - - - - - + - - - - - - - - - - - - - +            
                                 |
                       + --------------- +
                       | Bitcoin network |  <---  (3)   
                       + --------------- +        
        
        
 (1) You may connect an additinal node "Bob" and make the multihope 
 payment Alice->Faucet->Bob
  
 (2) "Faucet", "Alice" and "Bob" are the lightning network daemons which 
 creates the channel and interact with each other using Bitcoin network 
 as source of truth.
 
 (3) In current scenario "Alice" and "Faucet" lightning network nodes 
 connected to different Bitcoin nodes. If you decide to connect "Bob"
 to "Faucet" than already created "btcd" node would be sufficient.
```

First of all you need to run `btcd` node in `testnet` and wait it to be 
synced with test network (`May the Force and Patience be with you`).
```bash 
# Init bitcoin network env variable:
$ export NETWORK="testnet"

# Run "btcd" node:
$ docker-compose up -d "btcd"
```

After `btcd` synced, connect `Alice` to the `Faucet` node.
```bash 
# Run "Alice" container and log into it:
$ docker-compose up -d "alice"; docker exec -i -t "alice" bash

# Connect "Alice" to the "Faucet" node:
alice$ lncli connect <faucet_identity_address>@<faucet_host>
```

After connection was achieved the `Faucet` node should create the channel
and send some amount of bitcoins to `Alice`.

**What you may do next?:**
- Send some amount to `Faucet` node back.
- Connect `Bob` node to the `Faucet` and make multihop payment (`Alice->Faucet->Bob`)
- Close channel with `Faucet` and check the onchain balance.

### Questions
[![Irc](https://img.shields.io/badge/chat-on%20freenode-brightgreen.svg)]
(https://webchat.freenode.net/?channels=lnd)

* How to see `alice` | `bob` | `btcd` logs?
```bash
docker-compose logs <alice|bob|btcd>
```
