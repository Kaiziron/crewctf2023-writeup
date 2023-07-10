# CrewCTF 2023 - deception [12 solves / 957 points] [First blood 🩸]

Our goal is to set `solved` to true in the deception contract

### Setup.sol :
```solidity
pragma solidity ^0.8.13;

import "./Deception.sol";

contract Setup {
    deception public immutable TARGET;

    constructor() payable {
        TARGET = new deception(); 
    }

    function isSolved() public view returns (bool) {
        return TARGET.solved();
    }
}
```

### Deception.sol :
```solidity
// Contract that has to be displayed for challenge

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract deception{
    address private owner;
    bool public solved;

    constructor() {
      owner = msg.sender;
      solved = false;
    }

    modifier onlyOwner() {
      require(msg.sender==owner, "Only owner can access");
      _;
    }

    function changeOwner(address newOwner) onlyOwner public{
      owner = newOwner;
    }

    function password() onlyOwner public view returns(string memory){
        return "secret";
    }

    function solve(string memory secret) public {
      require(keccak256(abi.encodePacked(secret))==0x65462b0520ef7d3df61b9992ed3bea0c56ead753be7c8b3614e0ce01e4cac41b, "invalid");
      solved = true;
    }
}
```

We need to find the preimage of `0x65462b0520ef7d3df61b9992ed3bea0c56ead753be7c8b3614e0ce01e4cac41b`, there's a view function `password()` that return "secret"

So just hash it and we will see that "secret" is the preimage we need
```
➜ keccak256(abi.encodePacked("secret"))
Type: bytes32
└ Data: 0x65462b0520ef7d3df61b9992ed3bea0c56ead753be7c8b3614e0ce01e4cac41b
```

However, it reverts if I call solve() with "secret" :

```
# cast send 0xd92edc2A2cec7387d1bA68853f56B181e58f25Ee "solve(string)" "secret" -r http://146.148.125.86:60082/e317e7f0-6a27-4869-9cc9-e740bc6a1419 --private-key 0x7f425a14aa09dcc2d183ee3c6602dbe03eb919d1df05b8df26d82aea7888a44c
Error: 
(code: 3, message: execution reverted: invalid, data: Some(String("0x08c379a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000007696e76616c696400000000000000000000000000000000000000000000000000")))
```

As the name of this challenge is "deception", so I call `password()` on the deployed contract as the owner to see if it is the same as the source we have

```
# cast call 0xd92edc2A2cec7387d1bA68853f56B181e58f25Ee "password()(string)" -r http://146.148.125.86:60082/e317e7f0-6a27-4869-9cc9-e740bc6a1419 --from 0x3bb575846325074A559b1EFBAfEB5F623C30e811
xyzabc
```

Instead of "secret", it returns "xyzabc", so just call `solve()` with "xyzabc"

```
# cast send 0xd92edc2A2cec7387d1bA68853f56B181e58f25Ee "solve(string)" "xyzabc" -r http://146.148.125.86:60082/e317e7f0-6a27-4869-9cc9-e740bc6a1419 --private-key 0x7f425a14aa09dcc2d183ee3c6602dbe03eb919d1df05b8df26d82aea7888a44c

blockHash               0x23f7134f43f8029b4014735441f0cf30be28abbfe0ef50d84f6d416508ea93da
blockNumber             17656779
contractAddress         
cumulativeGasUsed       27514
effectiveGasPrice       3000000000
gasUsed                 27514
logs                    []
logsBloom               0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root                    
status                  1
transactionHash         0x5fe8f92b15f771145cc2c51840aaed9b568a15039549d3bc889bfbedc4bdc3e6
transactionIndex        0
type                    2
```

Then we can verify that we have solved the challenge :

```
# cast call 0x3bb575846325074A559b1EFBAfEB5F623C30e811 "isSolved()(bool)" -r http://146.148.125.86:60082/e317e7f0-6a27-4869-9cc9-e740bc6a1419
true
```

### Flag :

```
# nc deception.chal.crewc.tf 60002
1 - launch new instance
2 - kill instance
3 - get flag
action? 3
ticket please: REDACTED
This ticket is your TEAM SECRET. Do NOT SHARE IT!
crew{d0nt_tru5t_wh4t_y0u_s3e_4s5_50urc3!}
```