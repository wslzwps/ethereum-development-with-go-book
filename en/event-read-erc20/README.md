---
description: Tutorial on how to read ERC-20 Token smart contract events with Go.
---

# Reading ERC-20 Token Event Logs

Output

```bash
Log Block Number: 6383829
Log Index: 20
Log Name: Transfer
From: 0xd03dB9CF89A9b1f856a8E1650cFD78FAF2338eB2
To: 0xd03dB9CF89A9b1f856a8E1650cFD78FAF2338eB2
Tokens: 2804000000000000000000


Log Block Number: 6383831
Log Index: 62
Log Name: Approval
Token Owner: 0xDD3b9186Da521AbE707B48B8f805Fb3Cd5EEe0EE
Spender: 0xCf67d7A481CEEca0a77f658991A00366FED558F7
Tokens: 10000000000000000000000000000000000000000000000000000000000000000


Log Block Number: 6383838
Log Index: 13
Log Name: Transfer
From: 0xBA826fEc90CEFdf6706858E5FbaFcb27A290Fbe0
To: 0xBA826fEc90CEFdf6706858E5FbaFcb27A290Fbe0
Tokens: 2863452144424379687066
```

---

### Full code

Commands

```bash
solc --abi erc20.sol
abigen --abi=erc20_sol_ERC20.abi --pkg=token --out=erc20.go
```

[erc20.sol](https://github.com/miguelmota/ethereum-development-with-go-book/blob/master/code/contracts_erc20/erc20.sol)

```solidity
pragma solidity ^0.4.24;

contract ERC20 {
    string public constant name = "";
    string public constant symbol = "";
    uint8 public constant decimals = 0;

    function totalSupply() public constant returns (uint);
    function balanceOf(address tokenOwner) public constant returns (uint balance);
    function allowance(address tokenOwner, address spender) public constant returns (uint remaining);
    function transfer(address to, uint tokens) public returns (bool success);
    function approve(address spender, uint tokens) public returns (bool success);
    function transferFrom(address from, address to, uint tokens) public returns (bool success);

    event Transfer(address indexed from, address indexed to, uint tokens);
    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);
}
```

[event_read_erc20.go](https://github.com/miguelmota/ethereum-development-with-go-book/blob/master/code/event_read_erc20.go)

```go
package main

import (
	"context"
	"fmt"
	"log"
	"math/big"
	"strings"

	token "./contracts_erc20" // for demo
	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

// LogTransfer ..
type LogTransfer struct {
	From   common.Address
	To     common.Address
	Tokens *big.Int
}

// LogApproval ..
type LogApproval struct {
	TokenOwner common.Address
	Spender    common.Address
	Tokens     *big.Int
}

func main() {
	client, err := ethclient.Dial("https://mainnet.infura.io")
	if err != nil {
		log.Fatal(err)
	}

	// 0x Protocol (ZRX) token address
	contractAddress := common.HexToAddress("0xe41d2489571d322189246dafa5ebde1f4699f498")
	query := ethereum.FilterQuery{
		FromBlock: big.NewInt(6383820),
		ToBlock:   big.NewInt(6383840),
		Addresses: []common.Address{
			contractAddress,
		},
	}

	logs, err := client.FilterLogs(context.Background(), query)
	if err != nil {
		log.Fatal(err)
	}

	contractAbi, err := abi.JSON(strings.NewReader(string(token.TokenABI)))
	if err != nil {
		log.Fatal(err)
	}

	logTransferSig := []byte("Transfer(address,address,uint256)")
	LogApprovalSig := []byte("Approval(address,address,uint256)")
	logTransferSigHash := crypto.Keccak256Hash(logTransferSig)
	logApprovalSigHash := crypto.Keccak256Hash(LogApprovalSig)

	for _, vLog := range logs {
		fmt.Printf("Log Block Number: %d\n", vLog.BlockNumber)
		fmt.Printf("Log Index: %d\n", vLog.Index)

		switch vLog.Topics[0].Hex() {
		case logTransferSigHash.Hex():
			fmt.Printf("Log Name: Transfer\n")

			var transferEvent LogTransfer

			err := contractAbi.Unpack(&transferEvent, "Transfer", vLog.Data)
			if err != nil {
				log.Fatal(err)
			}

			transferEvent.From = common.HexToAddress(vLog.Topics[1].Hex())
			transferEvent.To = common.HexToAddress(vLog.Topics[2].Hex())

			fmt.Printf("From: %s\n", transferEvent.From.Hex())
			fmt.Printf("To: %s\n", transferEvent.From.Hex())
			fmt.Printf("Tokens: %s\n", transferEvent.Tokens.String())

		case logApprovalSigHash.Hex():
			fmt.Printf("Log Name: Approval\n")

			var approvalEvent LogApproval

			err := contractAbi.Unpack(&approvalEvent, "Approval", vLog.Data)
			if err != nil {
				log.Fatal(err)
			}

			approvalEvent.TokenOwner = common.HexToAddress(vLog.Topics[1].Hex())
			approvalEvent.Spender = common.HexToAddress(vLog.Topics[2].Hex())

			fmt.Printf("Token Owner: %s\n", approvalEvent.TokenOwner.Hex())
			fmt.Printf("Spender: %s\n", approvalEvent.Spender.Hex())
			fmt.Printf("Tokens: %s\n", approvalEvent.Tokens.String())
		}

		fmt.Printf("\n\n")
	}
}
```

solc version used for these examples

```bash
$ solc --version
0.4.24+commit.e67f0147.Emscripten.clang
```