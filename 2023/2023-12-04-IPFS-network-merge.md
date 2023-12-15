# 2023-12-04 IPFS network merge

## Authors

[@sh3ll3x3c](https://github.com/sh3ll3x3c)

## Status

Most immediate remedies have been deployed on the Avail Goldberg testnet. Monitoring the situation.

## Summary

Kademlia Distributed Hash Table (DHT) tables of IPFS (Amino) and Avail light client (LC) network were inadvertently merged, causing a significant network performance degradation both on the Avail LC P2P network and IPFS.

## Impact

Avail light client P2P network suffered a significant network degradation measured in overall increased latency for both `get` and `put` Kademlia operations, frequent query timeouts, and prolonged bootstrap process, all of which resulted in decreased DHT random sampling hit rate which is its primary purpose.

On the other side, IPFS Amino DHT suffered increased latency across the board, with more details found in the incident [report](https://discuss.ipfs.tech/t/incident-report-increased-latency-on-the-amino-dht/17338).

## Root cause

The main root cause of the incident was the improper Kademlia protocol name filtering, which was most likely inadvertently removed from Avail LC in one of the previous rust-libp2p upgrades. Kademlia protocol name is used to enforce peer and network separation between different DHTs, with a default library value falling back to that of IPFSs (`/ipfs/kad/1.0.0`), which was in this case also used by the Avail LC network.

This, in tandem with automatic peer address addition to the Kademlia routing table on newly received `identify` events enabled external (IPFS) peers to be introduced to the Avail network and the other way around.

## Trigger

The root cause was triggered by a peer that "bridged" the networks.

The most likely trigger was a single peer that connected to both the IPFS Amino and Avail DHTs, introducing the first IPFS peers to Avail and vice versa. 

Routing tables of those peers started to get “polluted” with outside peers in turn resolving Kademlia queries on the “wrong” outside peers, sending data from one network to the other. Periodic bootstrap implementation together with the identify protocol probably hastened the peer discovery, as it should, which led to a quicker network merge.

## Detection

The first issues on the Avail LC network were discovered on the Avail LC dashboard last week of November, with the sharp decrease in DHT hit rates coupled with a sharp increase in query latency, bootstrap failures, and overall network degradation. 

The Probelab team confirmed our suspicions a week later with their Nebula crawler, which detected issues both on IPFS and Avail LC networks.

## Timeline {WIP}

The first debugging efforts were undertaken immediately on the network degradation discovery in late November. A ping scan client was deployed to pinpoint problematic peers and allow us to create an initial landscape of the problem. At least a couple of dozens of peers were detected with ping latencies >20 seconds, all of which were peers that were deployed outside Avail's internal testing network. At that point, we still had no idea that the networks were merged and that the outside peers were non-Avail peers.

Efforts were first taken to offload the synchronous data upload (`put`) operation which caused the light and fat clients to fall out of sync with the Avail network, which coincided with the increase in block sizes on Avail. The team's initial thoughts were that the network degradation must have been caused by the increase in the amount of data uploaded due to the block size increase.

After this effort was done, the team saw that the network degradation persisted, even though it was briefly ameliorated due to LCs being redeployed and their in-memory Kademlia routing tables cleared.

Once the performance issues came back, LC P2P event loop tracing logs were used to try to figure out what exactly was happening, and that’s when we found that a huge number of peers were contacting Avail clients, with non-Avail protocol names.

The first week of December was when the Probelab team let us know their insights, which finally confirmed what we suspected about the outside peers interfering with the LC network.

# Action items

Three main action items resulted from this incident:

1. Non-Avail clients were to be filtered on received `identify` events, filtering them out and disabling incoming connections through the use of `allow-block-list` on the Swarm level.
2. Re-introducing the Kademlia protocol names. 
3. If steps 1. and 2. do not result in a decrease in Avail LC network query latencies, the P2P event loop will be analyzed for additional causes of the slowdowns.
4. Additional observability was introduced for p2p network monitoring

