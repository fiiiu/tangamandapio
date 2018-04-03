Multiple inheritance is hard, and it can get very confusing if you abuse it.
Some languages don't support it at all; but it is becoming a very common
pattern on Solidity. Thus, one of the first things to do when starting to
learn Solidity is to fully understand how it handles multiple inheritance.

The problem, from the point of view of the programmer, is to understand the
order of the calls when a contract has multiple parents. Let's say we have a
base contract `A`, with a function named `f`:

```
// in A.
function f() {
  somethingA();
}
```

Then we have three contracts `B`, `C`, and `D`, all of them overriding `f`. `B`
does something like this:

```
// in B.
function f() {
  somethingB();
  super.f();
}
```

`C` does something like this:

```
// in C.
function f() {
  somethingC();
  super.f();  
}
```

And `D` does something like this:

```
// in D.
function f() {
  somethingD();
  super.f();  
}
```

What happens when you call `D.f`?

This is called the diamond problem, because we end up with a diamond-shaped
inheritance diagram:

[![Diamond inheritance problem](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8e/Diamond_inheritance.svg/180px-Diamond_inheritance.svg.png)](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8e/Diamond_inheritance.svg/180px-Diamond_inheritance.svg.png)

(the image is from wikipedia)

To solve it, Solidity (just like Python, luckily for me) uses C3 linearization,
also called Method Resolution Order (MRO). This means that it will linearize
the inheritance graph. If `D` is defined like:

```
contract D is B, C {}
```

then `D.f` will call:

```
somethingD();
somethingB();
somethingC();
somethingA();
```

When `D` inherits from `B, C`, the linearization results in D→ B→ C→ A. This
means that `super` on `D` calls `B`. And you might be a little surprised by the
fact that calling `super` on `B` will result on a call to `C` instead of `A`,
even though `B` doesn't inherit from `C`. Finally, `super` on `C` will call
`A`.

If `D` is defined like:

```
contract D is C, B {}
```

then `D.f` will call:

```
somethingD();
somethingC();
somethingB();
somethingA();
```

When `D` inherits from `C, B`, the linearization results in D→ C→ B→ A.

Notice here that the order in which you declare the parent contracts matters a
lot. If the inheritance graph is not too complex, it will be easy to see that
the order of calls will follow the order in which the parents were declared,
left to right. If your inheritance is more complicated than this and the
hierarchy is not very clear, you should probably stop using multiple
inheritance and look for an alternate solution.

I recommend you to read the wikipedia pages of
[Multiple inheritance](https://en.wikipedia.org/wiki/Multiple_inheritance) and
[C3 linearization](https://en.wikipedia.org/wiki/C3_linearization), and the
[Solidity docs about multiple inheritance](https://solidity.readthedocs.io/en/latest/contracts.html#multiple-inheritance-and-linearization).
You will find in there a complete explanation of the C3 algorithm, and an
example with a more complicated inheritance graph.

After some time [auditing smart contracts](https://zeppelin.solutions/security-audits),
and [supporting the OpenZeppelin framework](https://openzeppelin.org/), the
Zeppelin team started noticing security vulnerabilities coming from a
misunderstanding of how Solidity linearizes multiple inheritance. In
particular, I would like to take a look at the case that Philip Daian
brilliantly explains on his blog post
[Solidity anti-patterns: Fun with inheritance DAG abuse](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/).

In there, he presents a Crowdsale contract that needs "to have a whitelist 
pool of preferred investors able to buy in the pre-sale [...], along with a 
hard cap of the number of [...] tokens that can be distributed." You can look
at the
[code for this crowdsale](https://github.com/Arachnid/uscc/tree/master/submissions-2017/philipdaian/crowdsale),
but let me try to summarize the scenario in pseudocode, with some
simplifications. There is a `Crowdsale` base contract with a `validate`
condition `hasStarted() && !hasEnded()`. There is also a `CappedCrowdsale`
contract that inherits from `Crowdsale` with the `validate` condition
`super.validate() && withinCap(msg.value)`. And there is a
`PreSaleWhitelistedCrowdsale` contract that also inherits from `Crowdsale` with
the validate condition
`super.validate() || (isInWhitelist(msg.sender) && !hasEnded)`.

Of course, we will use multiple inheritance to form the "deadly diamond of 
death", and we start to get nervous because now it sounds veeery risky :)

Let's define the
`contract PreSaleWithCapCrowdsale is PreSaleWhitelistedCrowdsale, CappedCrowdsale`,
From what we learned before, this will result in the linearization
`PreSaleWithCapCrowdsale`→ `PreSaleWhitelistedCrowdsale`→ `CappedCrowdsale`→ `Crowdsale`,
and thus, the `validate` condition of `PreSaleWithCapCrowdsale` will be:

```
((hasStarted() && !hasEnded()) && withinCap(msg.value)) || (isInWhitelist(msg.sender) && !hasEnded())
```

Pay close attention to the resulting condition, and notice that if a
whitelisted investor buys token before the crowdsale has ended, she will be
able to bypass the hard cap, and buy as many tokens as she wants.

Let's fix it, just by swapping the order on
`contract PreSaleWithCapCrowdsale is CappedCrowdsale, PreSaleWhitelistedCrowdsale`.
which will give us the linearization
`PreSaleWithCapCrowdsale`→ `CappedCrowdsale`→ `PreSaleWhitelistedCrowdsale`→ `Crowdsale`,

and the `validate` condition will be:

```
((hasStarted() && !hasEnded()) || (isInWhitelist(msg.sender) && !hasEnded())) && withinCap(msg.value)
```

Now, if the purchase attempt goes above the cap, the transaction will be
reverted even if the sender is whitelisted.

It's very easy to think that the parent contracts will just be magically merged
into something that will make sense to our use case, or to make a mistake when
we linearize them in our mind. Every use case will be different, so our
framework can't save you from the work of organizing your contracts' hierarchy;
but, we can make our contracts clearer and make sure that they will just work
for simple cases.

Since `zeppelin-solidity 1.7` we have a suite of crowdsale contracts fully
rewritten with all of this in mind.

The base contract
[`Crowdsale` has a lot of comments on the source code](https://github.com/OpenZeppelin/zeppelin-solidity/blob/master/contracts/crowdsale/Crowdsale.sol).
It explains that some functions should not be overriden, like `buyTokens`. Some
others like `_preValidatePurchase` can be overriden to implement the
requirements of your crowdsale, but that extra behavior should be concatenated
with the one of the parent calling super, to preserve the validations from the
base contract. And, some functions like `_postValidatePurchase` can be just
added as hooks in other parts of the crowdsale's lifecycle.

In addition to that, we also provide some contracts for common crowdsale
scenarios involving
[distribution](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts/crowdsale/distribution),
[emission](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts/crowdsale/emission),
[price](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts/crowdsale/price),
and
[validation](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts/crowdsale/validation).
These contracts are very well tested, but as we have seen here, you should be
extra careful when combining them. Analyze the linearization of the inheritance
graph to get a clear view of the functions that will be called and their order,
and always always add a full suite of automated tests for your final crowdsale
contract to make sure that it enforces all the conditions.

We can confidently go a little beyond that. By now, you should have started to
suspect that the `PreSaleWhitelistedCrowdsale` we saw above became complicated
because of the `||` (or) condition. Things are a lot easier when all our
conditions are merged with `&&` (and), because on that case the order of the
conditions doesn't alter the result.

Let's say that we have a crowdsale that should only allow whitelisted
investors to buy tokens while the sale is open and the cap has not been
reached. In this case, the condition to check would be:

```
hasStarted() && !hasEnded() && isInWhiteList(msg.sender) && isWithinCap(msg.value)
```

Our contract would be as simple as:

```
pragma solidity ^0.4.18;

import "zeppelin-solidity/contracts/crowdsale/validation/TimedCrowdsale.sol";
import "zeppelin-solidity/contracts/crowdsale/validation/WhitelistedCrowdsale.sol";
import "zeppelin-solidity/contracts/crowdsale/validation/CappedCrowdsale.sol";

contract WhitelistedTimedCappedCrowdsale is TimedCrowdsale, WhitelistedCrowdsale, CappedCrowdsale {

  function WhitelistedTimedCappedCrowdsale(
    uint256 _rate,
    address _wallet,
    ERC20 _token,
    uint256 _openingTime,
    uint256 _closingTime,
    uint256 _cap
  )
    public
    Crowdsale(_rate, _wallet, _token)
    TimedCrowdsale(_openingTime, _closingTime)
    CappedCrowdsale(_cap)
    {
    }

}
```

It doesn't matter how you order the parents, all the conditions will be
checked always. But, take this with a grain of salt. If your parent contracts
are not super clear, they might be hiding an `||` condition in a few
hard-to-read code statements. And, all the conditions will be checked but the
order will be different. This can have different side-effects on Solidity, some
paths will execute statements that other paths won't, and it could lead an
attacker to find a specific path that is vulnerable.

To finish, I wanted to show you my repo where I will be doing
experiments with crowdsales and their tests:
[https://github.com/elopio/zeppelin-crowdsales](https://github.com/elopio/zeppelin-crowdsales)
In there you will see that on my
[PreSaleWithCapCrowdsale contract](https://github.com/elopio/zeppelin-crowdsales/blob/master/contracts/PreSaleWithCapCrowdsale.sol)
I preferred to be explicit about the conditions instead of using `super`:

```
function _preValidatePurchase(address _beneficiary, uint256 _weiAmount) internal {
  require(_beneficiary != address(0));
  require(_weiAmount != 0);
  require(block.timestamp >= openingTime || whitelist[_beneficiary]);
  require(block.timestamp <= closingTime);
  require(weiRaised.add(_weiAmount) <= cap);
}
```

I encourage you to join me on these experiments, to try different combinations
of crowdsales and to play switching the order of the inheritance graph. We will
learn more about the resulting code that Solidity compiles, we will improve our
mental picture of the execution stack, and we will practice writing the tests
that will fully cover all the possible paths. If you have questions, you find
errors on the implementations in OpenZeppelin, or you find an alternative
implementation that will make it easier to develop crowdsales on top of our
contracts, please
[let us know by sending a message to the slack channel](https://slack.openzeppelin.org/).
I am *elopio* in there.

Thanks a lot to [Ale](https://github.com/ajsantander) and
[Alejo](https://github.com/fiiiu), who worked on these new contracts and helped
me to understand them. <3

