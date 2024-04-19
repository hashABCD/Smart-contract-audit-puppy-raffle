### [M-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack, incrementing gas cost for the future entrants. 

**Description** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` array is, the more checks a new player will have to make. This means the gas costs for players who enters right when the raffle starts is dramatically lower than those who enter later. Every additional address in the `players` array, is an additional check the loop has to make. 

```javascript
// @audit Dos Attack
@>      for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

**Impact** The gas cost for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and causing a rush at hte start of a raffle to be one of the first entrants in the queue.

An attacker might make the `PuppyRaffle::players` array so big that no one else enters, guaranteeing themselves the win. 

**Proof of Concepts**
If we have two sets of 100 players enter, the gas costs will be as such:
- 1st 100 players: ~6,252,128 gas
- 2nd 100 players: ~18,068,215 gas
This is around 3x more expensive for the second 100 players.

<details>
<summary>PoC</summary>
Place the following test into `PuppyRaffleTest.t.sol`.

```javascript
function testDenialOfService() public {
        vm.txGasPrice(1);

        //Lets enter 100 players
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for(uint256 i=0; i<playersNum; i++){
            players[i] = address(i);
        }
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee*players.length}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedForFirst100 = (gasStart-gasEnd) * tx.gasprice;
        console.log("Gas cost for first 100 players : ", gasUsedForFirst100);

        //For second 100 players
        address[] memory players2 = new address[](playersNum);
        for(uint256 i; i<playersNum; i++){
            players2[i] = address(playersNum+i);
        }
        gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee*players2.length}(players2);
        gasEnd = gasleft();
        uint256 gasUsedForSecond100 = (gasStart-gasEnd) * tx.gasprice;
        console.log("Gas cost for secont 100 players : ", gasUsedForSecond100);

        assert(gasUsedForSecond100>gasUsedForFirst100);
    }
```
</details>

**Recommended mitigation** There are a few recomendations
1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a dublicate check doesn't prevent the saem person from entering multiple times, only the same wallet address. 
2. Consider using a mapping to check duplicates. This would allow constant time lookup of whether a user has already entered.

```diff

```
3. Alternatively chek Openzeppelin library ...