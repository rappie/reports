# Rounding errors in OUSD

## Introduction
This document describes the research and investigation of possible fixes for rounding errors in the `OUSD` contract.

The research is performed by Rappie (Twitter: @rappenstein2), between February and March of 2023.

## Scope
In this report, we will be looking at rounding errors in the `OUSD` contract. The main goal is to gain insight into the current behavior and limits of the system. Additionally, we will explore potential solutions.

We will start by providing some background on the occurrence of rounding errors in the `OUSD` contract. After this, we will utilize Echidna to detect them, and attempt to find suitable solutions on a per-case basis.

Finally, we will present our findings and try to provide reasonings as to why the rounding errors only seem to be solvable to a certain degree.

## About OUSD
OUSD is designed as a rebasing token. This means it is a type of token that algorithmically adjusts its supply based on certain conditions. In the case of OUSD, rebasing happens based on the passive yield generated by the protocol. This yield is being reflected in the user's balance steadily increasing over time.

Internally, OUSD uses a multiplier system for dynamically calculating user balances. Updating all accounts with every rebase would require too much gas. Instead, the balance consists of a credit amount and a (global) multiplier. When a rebase occurs, only the multiplier needs to be increased.

OUSD also supports users to opt in and out of rebasing. When a user decides to opt out, their current multiplier is locked. This locking is per account, resulting in many different multipliers being stored over time.

## Rounding errors in OUSD
Rounding errors occur in several parts of the `OUSD` contract. We will investigate some concrete examples below.

### Change supply
Let's start by looking at changes in supply. When a rebase happens, the `changeSupply` function is used to change the total supply of OUSD. This happens by increasing the `_rebasingCreditsPerToken` multiplier.

```solidity
_rebasingCreditsPerToken = _rebasingCredits.divPrecisely(
    _totalSupply.sub(nonRebasingSupply)
);

require(_rebasingCreditsPerToken > 0, "Invalid change in supply");

_totalSupply = _rebasingCredits
    .divPrecisely(_rebasingCreditsPerToken)
    .add(nonRebasingSupply);
```

Consider the following example:
1. User A has balance `1`
2. User B has balance `1`
3. Total supply is `2`
4. A rebase happens, changing total supply to `3`
5. User A & B will still have balance `1` because `1.5` gets rounded down to `1`
6. Total supply is now `3`

In this example, one of the basic invariants of ERC20 tokens is broken. The sum of all balances should always equal the total supply.

### Transfer
Rounding errors can occur when transferring tokens between a rebasing and a non-rebasing account. In this scenario, we are dealing with two distinct multipliers.

```solidity
function _creditsPerToken(address _account) internal view returns (uint256) {
    if (nonRebasingCreditsPerToken[_account] != 0) {
        return nonRebasingCreditsPerToken[_account];
    } else {
        return _rebasingCreditsPerToken;
    }
}
```

Due to working with two different multipliers, a slight difference between the credited amount for the recipient and the deducted amount for the sender can occur.

```solidity
uint256 creditsCredited = _value.mulTruncate(_creditsPerToken(_to));
uint256 creditsDeducted = _value.mulTruncate(_creditsPerToken(_from));
```

### Mint & Burn
Both minting and burning contain rounding problems. In this case, we will be focussing on `burn`.

```solidity
uint256 creditAmount = _amount.mulTruncate(_creditsPerToken(_account));
uint256 currentCredits = _creditBalances[_account];

// Remove the credits, burning rounding errors
if (
    currentCredits == creditAmount || currentCredits - 1 == creditAmount
) {
    // Handle dust from rounding
    _creditBalances[_account] = 0;
} else if (currentCredits > creditAmount) {
    _creditBalances[_account] = _creditBalances[_account].sub(
        creditAmount
    );
} else {
    revert("Remove exceeds balance");
}
```

As can be seen in the code above, the amount of balance to be burned needs to be converted to a number of credits to deduct from the user's credit balance.

Consider the following example:
1. User A has balance `100`
2. Total supply is `100`
3. Burn `1` token from user A
	- Amount `1` gets converted to `0` credits due to rounding down
	- `0` credits are subtracted from user A
	- `1` token is subtracted from the total supply
4. User A still has balance `100`
5. Total supply is now `99`

This effectively causes supply to vanish while the account balance stays the same. Thankfully, this scenario only works for `1` token at a time.

### Opt-in
Opting in can be seen as transferring all balance from an account in a non-rebasing state to the same account in rebasing state. Therefore, it suffers from the same kind of rounding errors as `transfer` does.

```solidity
uint256 oldBalance = balanceOf(msg.sender);
uint256 newCreditBalance = _creditBalances[msg.sender]
    .mul(_rebasingCreditsPerToken)
    .div(_creditsPerToken(msg.sender));
```

## Detecting rounding errors
We will be using Echidna to help us detect rounding errors. There are many strategies for this, we will keep things simple and use a single invariant.

This invariant states that the sum of all balances must be equal to the total supply.
```solidity
function invariantTotalBalanceEqualsTotalSupply() public {
    totalBalance = _getTotalBalance(); // sum all balances
    totalSupply = ousd.totalSupply();
    assert(totalBalance == totalSupply);
}
```

With `multi-abi` enabled, Echidna will be fuzzing the entire ABI of the contract. Additionally, we created helper functions to narrow the search space.

Example of helpers functions:
```solidity
function transfer(
    uint8 fromAccId,
    uint8 toAccId,
    uint256 amount
) public {
    address from = getAccountFromUint8(fromAccId);
    address to = getAccountFromUint8(toAccId);

    hevm.prank(from);
    ousd.transfer(to, amount);
}

function rebaseOptIn(uint8 targetAccId) public {
    address target = getAccountFromUint8(targetAccId);

    hevm.prank(target);
    ousd.rebaseOptIn();
}
```

We know there are existing rounding errors, so non-surprisingly, Echidna makes quick work of breaking our invariant. 

```solidity
invariantTotalBalanceEqualsTotalSupply(): failed!💥
  Call sequence:
    mint(<account1>,2)
    transfer(<account1>,<account2>,1)
    changeSupply(1)
    invariantTotalBalanceEqualsTotalSupply()

Event sequence: Panic(1): Using assert.
```

Our next step is to examine the error and try to fix this in the code as best we can. Our goal is to gain more insight into the reasons and possible solutions, so at the moment, this fix doesn't need to be perfect. Next, we run Echidna again.

We will keep doing this until either Echidna stops breaking the invariant or we reach a point where any more fixes are impossible or result in (unwanted) side effects.

Spoiler alert: Eventually we determined it to be impossible to get rid of all rounding errors (at least with our current understanding).

## Fixing rounding errors
We have investigated two different approaches to try and fix the rounding errors:
1. Tweaking functions in the current code
2. Storing rounding errors in a dedicated variable in the contract

Let's look at each approach in more detail below.

### Tweaking current code
In this approach, we will make small modifications to the existing functions and refrain from making any changes to the basic design of the contract.

#### Change supply
The current `changeSupply` function has known problems. There already exists an improved version of this in a pull request on Github. We will be using this newer version.

https://github.com/OriginProtocol/origin-dollar/blob/8f2df077b6652d48cbd56878c07116b3c56d01b9/contracts/contracts/token/OUSD.sol#L441-L494

#### Transfer
The main problem with `transfer` is the differences between the amount sent and the amount received. As explained earlier, this is due to rounding errors when using different "credits per token" multipliers.

In an attempt to fix this, we have explored the possibility to calculate the amount based on the multiplier with the lowest precision. This should help with making sure the amount transferred is the same on both ends.

```solidity
uint256 creditsCredited;
uint256 creditsDeducted;
if (toCpt == fromCpt) {
    creditsCredited = _value.mulTruncate(toCpt);
    creditsDeducted = _value.mulTruncate(fromCpt);
} else if (toCpt > fromCpt) {
    creditsDeducted = _value.mulTruncate(fromCpt);
    // deliberately do div before mul
    creditsCredited = creditsDeducted.divPrecisely(fromCpt).mulTruncate(
            toCpt
        );
} else {
    creditsCredited = _value.mulTruncate(toCpt);
    // deliberately do div before mul
    creditsDeducted = creditsCredited.divPrecisely(toCpt).mulTruncate(
        fromCpt
    );
}
```

This works to an extent but opens up a broader discussion. For example: if you transfer amount X, you always want at least X tokens arriving at the sender. We will consider this discussion out of scope for the report.

#### Mint & Burn
We made several changes to `mint` and `burn`. The changes are similar in nature, so we will only be describing the `burn` function here.

For starters, we added a `require` statement that reverts when the number of tokens to be burned translates to `0` credits.

```solidity
require(creditAmount > 0, "Burn amount too small");
```

Second, we removed some of the existing code meant to reduce the possibility of "dust" remaining after a burn.

```solidity
if (
    currentCredits == creditAmount || currentCredits - 1 == creditAmount
) {
    // Handle dust from rounding
    _creditBalances[_account] = 0;
} else if (currentCredits > creditAmount) {
    _creditBalances[_account] = _creditBalances[_account].sub(
        creditAmount
    );
} else {
    revert("Remove exceeds balance");
}
```

Finally, we subtract the actual change in balance from `nonRebasingSupply` instead of relying on the `_amount` argument.

```solidity
if (isNonRebasingAccount) {
    uint256 balanceChange = creditAmount.divPrecisely(
        _creditsPerToken(_account)
    );
    nonRebasingSupply = nonRebasingSupply.add(balanceChange);
} else {
    _rebasingCredits = _rebasingCredits.add(creditAmount);
}
```

#### Opt-in
As mentioned above, opting in is basically a self-transfer from a non-rebasing to a rebasing account. In contrast to `transfer`, we are dealing with just one account here. This means that we cannot easily guarantee the balance to remain the same.

We chose to update the total supply in the case of a change in balance.

```solidity
if (newBalance < oldBalance) {
    _totalSupply = _totalSupply.sub(oldBalance.sub(newBalance));
} else if (newBalance > oldBalance) {
    _totalSupply = _totalSupply.add(newBalance.sub(oldBalance));
} else {
    // do nothing
}
```

This again opens up a broader discussion. The total supply is managed by the `Vault` contract, and should only be changed in the case of a rebase. We consider this discussion to be out of the scope of this report.

### Storing rounding errors
The reasoning behind this approach is as follows: we know there are rounding errors, why not embrace them instead of trying to get rid of them? What if we can determine the error being introduced with each action and store this in a separate signed integer variable?

Spoiler alert: In the end, it turned out we can't always determine the error being introduced. More on this later.

#### Variable storing rounding errors
We start by introducing a new signed integer variable. This variable will be updated when any rounding error occurs.

```solidity
int256 totalSupplyRoundingError;
```

#### Total supply
Next, we need our total supply to take any accrued rounding errors into account.

```solidity
function totalSupply() public view override returns (uint256) {
    int256 totalSupply = int256(_totalSupply) + totalSupplyRoundingError;
    if (totalSupply < 0) {
        return 0;
    }
    return uint256(totalSupply);
}
```

#### Transfer
To determine if transferring tokens causes any rounding errors, we look at the differences in balance before and after the transfer.

```solidity
uint256 fromBalanceBefore = balanceOf(_from);
uint256 toBalanceBefore = balanceOf(_to);

_creditBalances[_from] = _creditBalances[_from].sub(
    creditsDeducted,
    "Transfer amount exceeds balance"
);
_creditBalances[_to] = _creditBalances[_to].add(creditsCredited);

uint256 fromBalanceAfter = balanceOf(_from);
uint256 toBalanceAfter = balanceOf(_to);

int256 fromDelta = int256(fromBalanceAfter) - int256(fromBalanceBefore);
int256 toDelta = int256(toBalanceAfter) - int256(toBalanceBefore);
```

In case of rounding errors, we update `totalSupplyRoundingError` accordingly.

```solidity
int256 roundingError = fromDelta + toDelta;
totalSupplyRoundingError = totalSupplyRoundingError + roundingError;
```

#### Mint
For minting, we do the same. We store the balance before and after minting and store any rounding errors.

```solidity
uint256 balanceBefore = balanceOf(_account);
_creditBalances[_account] = _creditBalances[_account].add(creditAmount);
uint256 balanceAfter = balanceOf(_account);

...

uint256 balanceIncrease = balanceAfter.sub(balanceBefore);
int256 roundingError = int256(_amount) - int256(balanceIncrease);

totalSupplyRoundingError = totalSupplyRoundingError - roundingError;
```

#### Change supply
While being led by Echidna to find the next error after each modification we made, we ended up taking a look at `changeSupply`. This is where we hit a roadblock.

To update `totalSupplyRoundingError`, we need to determine if any rounding errors are being introduced during the execution of `changeSupply`. For this, we would have to compare the sum of all balances after the change with the new total supply.

This is not possible, because we would have to iterate over all accounts, which would cost way too much gas. The `_rebasingCreditsPerToken` variable is used specifically to prevent these kinds of iterations from being necessary.

Without being able to determine the rounding error here, it was concluded that this approach is not feasible.

In the next section, we will describe any new insights gained from attempting this approach.

## Conclusion
Dealing with rounding errors in Solidity is hard.

During this project, we had the luxury of leaving broader discussions about the intended behavior of OUSD out of scope. In reality, we've come to learn, dealing with rounding errors is much more complex than just balancing accounts and variables.

With the current setup of OUSD and the existing feature set of Solidity, it appears that rounding errors are here to stay for years to come. Despite this, improvements can be made.

Even though our effort didn't succeed in removing all rounding errors, some can be prevented. It might be worthwhile to examine this further in the future.

Some considerations:
- Revert when minting/burning an amount that would result in `0` credits
- Move away from the total supply having to remain static between rebases. This will enable the possibility to do rewrites of OUSD in the future, e.g. dynamically calculate the total supply
- Setting limits on `_rebasingCreditsPerToken` to prevent extreme situations and have well-defined limits

Additionally, we have identified several strategies to reduce the impact of rounding errors on the end user.
- Using higher precision (more decimals) internally reduces the amount of rounding errors visible from the outside
- Make sure that during a transfer the amount sent is rounded down and the amount received is rounded up
- Make sure opting in does not reduce the balance of the user

## Future research
We believe that having someone with a background in mathematics take a look at the core problems behind rounding errors would be beneficial.

Another approach would be to re-design the whole system from scratch while taking advantage of our current knowledge. This new design doesn't necessarily need to be implemented to be of use. It could shed new light on current problems and provide guidance on how to incrementally work towards a future goal.

Knowing the current limitations and requirements of the protocol as a whole also helps. As first steps one could consider:
1. Adding fuzzing to the test suite to gain more understanding of edge cases
2. Writing user stories for common actions to better define the intended behavior

## References & Acknowledgements
Origin Dollar Pull Request *"Rethink changeSupply"*
- https://github.com/OriginProtocol/origin-dollar/pull/561

Echidna - Smart Contract Fuzzer
- https://github.com/crytic/echidna

ChatGPT - Utilized as a resource to assist in writing this report
- https://chat.openai.com/chat

Many thanks to DanielVF for the support and feedback

## Disclaimer
The information, advice, or services provided by me (Rappie) were based on my best knowledge and abilities at the time of delivery. However, I cannot guarantee the accuracy, completeness, or usefulness of the information provided, nor can I be held liable for any damages or losses resulting from the use or reliance on such information or services. Any actions taken based on the information or advice provided are done so at the sole discretion and risk of the individual or organization involved. This disclaimer serves to clarify that while I strive to provide helpful and accurate information or services, there are no legal guarantees or warranties provided.

