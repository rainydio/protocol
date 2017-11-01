# Optimization Bounty Submissions

We'll be collecting optimization bounty submissions and their responses here. Please be sure to take a look before making a submission. Thank you!

## #01 [merged]

- From: Brecht Devos <brechtp.devos@gmail.com>
- Time: 11:22 29/10/2017 Beijing Time
- PR: https://github.com/Loopring/protocol/pull/35
- Result: This PR simplies the code but doesn't reduce gas usage. We encourage Brecht to confirm our findings.

Hey,
 
I think I significantly reduced gas usage by optimizing the xorOp function in Bytes32Lib. This function is used in calculateRinghash. This is how I updated the function:
 
```
function xorOp(
        bytes32 bs1,
        bytes32 bs2
        )
        internal
        constant
        returns (bytes32 res)
    {
        uint temp = uint(bs1) ^ uint(bs2);
        res = bytes32(temp);
    }
 ```
 
The original code seems to make things more difficult than needed though there could be reasons for that that I don’t know.
This change reduces the gas usage a nice ~10% in the 3 order ring test in my quick measurements I did. I will do more exhaustive testing of this change when I have a bit more time, but I didn’t want to wait too long before submitting this for a possible bounty. 😊 All tests do seem to run just fine with this change.
 
If there any issues or additional questions please let me know.
 
Brecht Devos


## #02 [TBD]

- From: Kecheng Yue <yuekec@gmail.com>
- Time: 00:46 01/11/2017 Beijing Time
- PR: https://github.com/Loopring/protocol/pull/37
- Result: TBD

Hi,  

I made two optimizations for loopring protocol which reducing about 3.7% of the gas usage in the 3 orders ring test. All tests are passed for the changes.

1. Reduce the time complexity of `TokenRegistry.isTokenRegistered` from O(n) to O(1). `n` is the length of registered token list. This optimization is made for the reason that when `verifyTokensRegistered ` loops for each address in addressList, it calls `TokenRegistry.isTokenRegistered`, causing the time complexity is O(mn). It will be reduced to O(m) by this optimization. This change reduces the gas usage a ~2.2%.

```

contract TokenRegistry is Ownable {

    address[] public tokens;

    mapping (address => bool) tokenMap; // Add this.

    mapping (string => address) tokenSymbolMap;

    function registerToken(address _token, string _symbol)

        public

        onlyOwner

    {

        // ...

        tokens.push(_token);

        tokenMap[_token] = true; // Add this.

        tokenSymbolMap[_symbol] = _token;

    }

    // ... see details in TokenRegistry.sol

    function isTokenRegistered(address _token)

        public

        constant

        returns (bool)

    {

        return tokenMap[_token]; // Add this.


    }

```

2. Optimize `ErrorLib.check` usage in many places in the codes. In fact, I just inline the function in some elementary operations. That is `ErrorLib.check(condition, message) => If (!condition) {ErrorLib.error(message)}`. Given that, ErrorLib.check is called in some elementary operations, whatever the value of `condition` is, making the function inline will avoid the additional cost for calling function. Certainly, `real` inline funcations are commonly supported by compiler. But as I know, inline functions are planned but not yet supported by official for now (http://solidity.readthedocs.io/en/v0.4.15/types.html). This change reduces the gas usage a ~2.2%.

See details in LoopringProtocolImpl.sol





因为不擅长用英语写文章，所以以下用中文描述了一遍。

Hi,

我对loopring protocal做了2点优化，使得`3 order ring test` 减少了大约3.7%gas使用量。测试用例已通过。

1 将TokenRegistry.isTokenRegistered的时间复杂度从O(n)改到O(1)(n为RegisteredToken.length)。之所以做这样的优化，是因为verifyTokensRegistered方法在遍历addressList时，调用了TokenRegistry.isTokenRegistered。优化后可以将复杂度从O(mn)降为O(m)。此项优化大约减少了2.2%左右的gas消耗。

代码见英文处

2 对多处使用到ErrorLib.check的地方做了优化。其实就是将其inline化。即：ErrorLib.check(condition, message) => If (!condition) {ErrorLib.error(message)}。之所以这么做是因为考虑到ErrorLib.check出现在了很多关键操作中，并且无论condition为何值，都会引起一个函数调用，将其inline化可以避免函数调用所引起的额外消耗。当然，这样的inline化通常是交给编译器来进行的，不过就目前为止，inline function 并未被支持（但已在计划中）。此项优化大约减少了1.5%左右的gas消耗。考虑到以后inline function可能会被官方支持，并且此优化所带来的改进较小，是否需要如此优化值得商榷。

代码见附件

## #03 [TBD]

- From: Brecht Devos <brechtp.devos@gmail.com>
- Time: 00:55 01/11/2017 Beijing Time
- PR: TBD
- Result: TBD

Hi,
 
I reduced the number of necessary SSTORE (and SLOAD) instructions in submitRing. The idea is pretty simple: 2 storage variables are always updated in the function: ‘entered’ and ‘ringIndex’.
'entered’ is used to check for reentrancy of the function, so it’s updated once at the very beginning and once at the very end (2 SSTORES). ‘ringIndex’ is also read in the function and updated at the end (1 SSTORE).
You can reduce the number of SSTORES by combining these 2 storage variables in 1. Instead of setting ‘entered’ to true at the beginning, you can set ‘ringIndex’ to an invalid value (uint(-1)). So the reentrance check becomes ‘ringIndex != uint(-1)’.
At the end of the function ‘ringIndex’ is updated again with it’s original value incremented by 1. This also signals that the function has reached its end (‘ringIndex’ != uint(-1)). This is where the SSTORE instruction is saved, before the change 2 SSTORE instructions were needed to update ‘entered’ and ‘ringIndex’.
 
Some thoughts about the change:
Reading the storage variable ‘ringIndex’ while submitRing is running will not return the correct value (as it is set to uint(-1)). This shouldn’t be a problem because (as far as I know) this can only be done in a reentrance scenario.
But this still could be fixed by reserving a single bit of ‘ringIndex’ as a sort of ‘busy bit’. This bit could be set at the start of the function (‘ringIndex |= (1 << 255)’) without destroying the actual index bits. The actual ‘ringIndex’ could then be read by ignoring the busy bit.
Extra care needs to be given to not accedentially read from the ‘ringIndex’ storage variable in the submitRing function. This isn’t that big of a problem because it’s used only twice.
 
This change saves a bit more than 1% in gas (which is what I expected calculating the theoretical gas costs).
 
Let me know what you think of this optimization. For completeness’ sake I pasted the git diff below with all necessary changes. If you’re alright with the change I could make a pull request if you want.
I had to put the calculateRinghash inside its own function to save on local variables inside submitRing(). Otherwise it’s some very small changes in a couple of places.
 
Brecht Devos