# Steps to run bridge_network_erc20_test

```
#!/bin/bash
make build-docker-e2e-real_network

mkdir tmp

cat <<EOF > ./tmp/test.toml
TestAddrPrivate= "${{ SECRET_PRIVATE_ADDR }}"
[ConnectionConfig]
L1NodeURL="${{ SECRET_L1URL }}"
L2NodeURL="${{ L2URL }}"
BridgeURL="${{ BRIDGEURL }}"
L1BridgeAddr="${{ BRIDGE_ADDR_L1 }}"
L2BridgeAddr="${{ BRIDGE_ADDR_L2 }}"
EOF

docker run  --volume "./tmp/:/config/" --env BRIDGE_TEST_CONFIG_FILE=/config/test.toml bridge-e2e-realnetwork-erc20
```

test.toml (localhost)
----------------------

```
TestL1AddrPrivate="0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
TestL2AddrPrivate="0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"

[ConnectionConfig]
L1NodeURL="http://localhost:8545"
L2NodeURL="http://localhost:8123"
BridgeURL="http://localhost:8080"
L1BridgeAddr="0xFe12ABaa190Ef0c8638Ee0ba9F828BF41368Ca0E"
L2BridgeAddr="0xFe12ABaa190Ef0c8638Ee0ba9F828BF41368Ca0E"
```

test.toml (testnet)
--------------------

```
TestAddrPrivate=""

[ConnectionConfig]
L1NodeURL="https://eth-sepolia.g.alchemy.com/v2/5tbOoJHN7gE_YehMETdquf80FE7TlYn_"
L2NodeURL="https://rpc.cardona.zkevm-rpc.com"
BridgeURL="https://bridge-api.cardona.zkevm-rpc.com"
L1BridgeAddr="0x528e26b25a34a4a5d0dbda1d57d318153d2ed582"
L2BridgeAddr="0x528e26b25a34a4a5d0dbda1d57d318153d2ed582"
```

# Localhost Error

**Error auth from L1 test:  Err:Post "http://localhost:8545": dial tcp [::1]:8545: connect: connection refused** (geth)    
**Error auth from L1 test:  Err:Post "http://localhost:8123": dial tcp [::1]:8123: connect: connection refused** (zkevm-node)

note :
-----
hermeznetwork/geth-zkevm-contracts:elderberry-fork.9-geth1.13.11 is running in the port 8545.  
Getting response from 8545 port.

# Testnet Error
Error: **deposit.ReadyForClaim = false**

**deposit.ReadyForClaim** boolean value is used to claim the token in destination chain.
As it doesn't turns true, the claim is not getting executed.
ReadyForClaim boolean cannot be manually changed.

note : 
-----
ERC20 contract is deployed on testnet.     
Token is transferred to L1BridgeAddr and returns a deposit hash.


```
func waitDepositToBeReadyToClaim(ctx context.Context, testData *bridge2e2TestData, asssetTxHash common.Hash, timeout time.Duration, destAddr string) (*pb.Deposit, error) {
	startTime := time.Now()
	if len(destAddr) < 1 {
		destAddr = testData.auth[operations.L1].From.String()
	}
	fmt.Println("test debug -> destAddr : ", destAddr)
	for true {
		fmt.Println("Waiting to deposit (", destAddr, ") fo assetTx: ", asssetTxHash.Hex(), "...")
		deposits, _, err := testData.l1BridgeService.GetBridges(destAddr, 0, 100)
		if err != nil {
			return nil, err
		}

		for _, deposit := range deposits {
			depositHash := common.HexToHash(deposit.TxHash)
			// fmt.Println()
			fmt.Println("test debug 2 : ", depositHash)
			fmt.Println("test debug 2.1 : ", deposit.ClaimTxHash)
			if depositHash == asssetTxHash {
				fmt.Println("Deposit found: ", deposit, " ready:", deposit.ReadyForClaim)

				if deposit.ReadyForClaim { //test
					// if true {
					fmt.Println("Found claim! Claim Is ready  Elapsed time: ", time.Since(startTime))
					return deposit, nil
				}
			}
		}
		if time.Since(startTime) > timeout {
			// if true {
			return nil, fmt.Errorf("Timeout waiting for deposit to be ready to be claimed")
		}
		fmt.Println("Sleeping ", timeBetweenCheckClaim.String(), "Elapsed time: ", time.Since(startTime))
		time.Sleep(timeBetweenCheckClaim)
	}
	return nil, nil
}
```


