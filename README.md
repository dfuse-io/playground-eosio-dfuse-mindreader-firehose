## Playground - Generate dfuse Block using Mindreader and start dfuse Firehose

In this playground, we show case how you can start syncing a dfuse Mindreader with your network
in a process and start the dfuse Firehose app that consumes those blocks to offer a stream of blocks.

We will sync with Telos to showcase a low/medium traffic chain, syncing EOS Mainnet would require
more horsepower than necessary for this guide.

What do you need to start:
- dfuse Instrumented `nodeos` (deep-mind) binary (https://github.com/dfuse-io/dfuse-eosio/blob/develop/DEPENDENCIES.md#dfuse-instrumented-eosio-prebuilt-binaries)
- Latest version of dfuse for EOSIO binary (https://github.com/dfuse-io/dfuse-eosio#from-source)
- An Telos snapshot that can be found here: https://snapshots.eosamsterdam.net/public/telos/, see [Snapshots Section](#snapshots) for instructions.

**Note** The pre-built binary (https://github.com/dfuse-io/dfuse-eosio/releases) could work, it's just
not the one that was tested.

Ensure that both are working are correclty installed/available with the following commands:

```bash
# The output version should have `dm` in it, it it does not, this is not a dfuse Instrumented `nodeos` process
$ nodeos --version
v2.0.8-dm.12.0

$ dfuseeos --version
dfuseeos version 0.1.0-beta8-437fe37d92afafcbcdc2efb850162f3372673bf0
```

If one of the two command fails, check back the instructions or seek for support.

### In This Guide

We are going to run a dfuse Mindreader app that manages `nodeos` (start it, controls it, etc) and that
automatically produces dfuse Blocks. We will then consume those dfuse Blocks to offer a dfuse Firehose
streams of those.

#### Mindreader

Let's first start dfuse Mindreader to generate merged blocks. Prior that, edit the `mindreader.yaml`
config and find the `` line, update it to point to the name of the snapshot file your downloaded.

Then run

```bash
dfuseeos start -c mindreader.yaml -d run/mindreader -v
```

This start dfuse for EOSIO, tells it to use the `mindreader.yaml` config file and use `run/mindreader` as
it's transient data directory. The config tells dfuse Mindreader where to write merged block (in `./blocks/merged`).

It will take a 2 to 3 minutes to start syncing, let it run something like 10m, you can monitor the `./blocks/merged`
file for generated dfuse Blocks:

```bash
# After running ~7m
ls blocks/merged | wc -l
     229

du -sh blocks/merged
 85M	blocks/merged
```

At that point, we had 229 files (so 22 900 blocks (229 * 100)) generated, if you let it run long enough, it
will eventually catch up and stay in sync with the chain.

When you have a couple thousand of blocks available, note the start and end block you have, we will use that
in the next step to test bench dfuse Firehose.

```
ls blocks/merged | sort | head -n1
0127365600.dbin.zst

ls blocks/merged | sort | tail -n1
0127388400.dbin.zst
```

Start block will #127 365 600 and end block will be #127 388 400.

#### Firehose

Now, you can start another dfuse for EOSIO process with a different config file and
different data directory.

This time, we are going to launch dfuse Firehose with the merged blocks that we have
generated so far, note that the other process could still be running, you will be able
to fetch a bigger range as blocks are generated.

```bash
dfuseeos start -c firehose.yaml -d run/firehose -v
```

**Note** You can run with `-vvv` to increase the verbosity level, usefull to see what
dfuse for EOSIO is trying to do.

You now have dfuse Firehose running a gRPC connection over TLS with a self-signed
certificate. Let's try to query a range of blocks.

We will query 400 blocks after the start block we noted, this is because some past
blocks must exist to fill our forking resolver algorithm.

Open another terminal an uses `grpcurl` to fetch the service.

```
grpcurl -insecure -d '{"start_block_num":"127366000","stop_block_num":127366001}' localhost:9000 dfuse.bstream.v1.BlockStreamV2/Blocks
Failed to format response message 1: unknown message type "dfuse.eosio.codec.v1.Block"
Failed to format response message 2: unknown message type "dfuse.eosio.codec.v1.Block"
```

Great, we have queried dfuse Firehose service and we receive something, you should have
seen some logs moving in your terminal where dfuse Firehose is started.

Now, why we get `unknown message type "dfuse.eosio.codec.v1.Block"`? Because dfuse Firehose
is a chain agnostic stream and you need to specify the kind of block we are expecting, here
it's a `dfuse.eosio.codec.v1.Block` message.

But `grpcurl` doesn't know about it, we would need to configure more and it was not easy.

Instead, let's clone https://github.com/dfuse-io/playground-firehose-eosio-go repository
and use it to retrieve the data instead.

```
cd /tmp
git clone https://github.com/dfuse-io/playground-firehose-eosio-go.git
cd playground-firehose-eosio-go

go run . -s -o - localhost:9000 "receiver in ['eosio']" "127 366 000 - 127 366 001"
```

**Note** The script is asking for a `DFUSE_API_KEY` environment here, it's not technically
required, your local endpoint is not authenticating. But the playground is not configurable
yet to connect to unauthenticated endpoint

You should see the following output:

```
go run . -s -o - localhost:9000 "receiver in ['eosio']" "127 366 000 - 127 366 001"
2021-01-15T19:34:07.963-0500 Starting firehose test {"endpoint": "localhost:9000", "range": "127366000 - 127366001"}
{"id":"079773705cc8035b7bc2512f2c59df0b8aec90d2877c208a855aceabf3544c93","number":127366000,"header":{"timestamp":"2020-12-21T16:16:28Z","producer":"goodblocktls"},"dposIrreversibleBlocknum":127365667,"filteredTransactionTraces":[{"id":"66831ca35befcfbcf54af4944d75394e100dd8ebd2e1e5b608cc83b2ccdcac40","receipt":{"status":"TRANSACTIONSTATUS_EXECUTED","cpuUsageMicroSeconds":100},"actionTraces":[{"receiver":"eosio","receipt":{"globalSequence":"6502297693"},"action":{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"jsonData":"{\"header\":{\"action_mroot\":\"a0c56811b33806e8effb6d2555701d9a20041ab5be174ccbec0098e6457527aa\",\"confirmed\":0,\"new_producers\":null,\"previous\":\"0797736ea8eaa4a31d762f3c1b35dcd6d693bf852bc612e3cbffd608112f6b90\",\"producer\":\"goodblocktls\",\"schedule_version\":2366,\"timestamp\":1323765175,\"transaction_mroot\":\"3dd8731544f9ce4ef5df935124052b8796157c2411d97d8d18a613b65cdf2bab\"}}"},"actionOrdinal":1}]}]}
{"id":"07977371aa9fd7f4e187dbb00ef2f2eca48ee34cce3ce6aaed173502ab8d30c1","number":127366001,"header":{"timestamp":"2020-12-21T16:16:28.500Z","producer":"goodblocktls"},"dposIrreversibleBlocknum":127365667,"filteredTransactionTraces":[{"id":"f0f1554665d007366a39750fac565ab11db0c4f64d34a77fbcb71fd21e3aaeb5","receipt":{"status":"TRANSACTIONSTATUS_EXECUTED","cpuUsageMicroSeconds":100},"actionTraces":[{"receiver":"eosio","receipt":{"globalSequence":"6502297694"},"action":{"account":"eosio","name":"onblock","authorization":[{"actor":"eosio","permission":"active"}],"jsonData":"{\"header\":{\"action_mroot\":\"6576e5e8c20d95728475c659895d0b820a114e14d71ee3b64a5717fac546a86e\",\"confirmed\":0,\"new_producers\":null,\"previous\":\"0797736fd82abd6f6ddacc920434199b61f6522f0b897e90cb75081cd7352eaf\",\"producer\":\"goodblocktls\",\"schedule_version\":2366,\"timestamp\":1323765176,\"transaction_mroot\":\"0000000000000000000000000000000000000000000000000000000000000000\"}}"},"actionOrdinal":1}]}]}

Completed streaming
Duration: 708.750033ms
Time to first block: 708.563712ms

Block received: 2 block/min (2 block total)
Bytes received: 1560 byte/min (1560 byte total)
```

#### Next Steps

The next steps will be to run everything using a live setup, TBC.

### Snapshots

Download the Telos snapshot, decompress it (you will need `zstd` binary) and move it in the
`snapshots` directory so it can be picked up correctly

```
# We assume you are at the root of this project, update link to latest snapshot
wget https://snapshots.eosamsterdam.net/public/telos/2020-12-21/telos_snapshot_127365532_0797719c55b9d42b7d00c54512bfa1e31be650c56060765ae7f93cfbf2c0b282.bin.zst

zstd -d telos_snapshot_127365532_0797719c55b9d42b7d00c54512bfa1e31be650c56060765ae7f93cfbf2c0b282.bin.zst

# Adjust the command to fit with the name of the latest snapshot
mv telos_snapshot_127365532_0797719c55b9d42b7d00c54512bfa1e31be650c56060765ae7f93cfbf2c0b282.bin snapshots/127365532_0797719c55b9d42b7d00c54512bfa1e31be650c56060765ae7f93cfbf2c0b282.bin

rm -rf telos_snapshot_127365532_0797719c55b9d42b7d00c54512bfa1e31be650c56060765ae7f93cfbf2c0b282.bin.zst
```