# Curta 17

This is a write-up for [Murder Mystery](https://www.curta.wtf/puzzle/17), that gives you several
clues towards solving it. I'll be adding more details to this writeup after phase 2 is finished. This doesn't fully explain the puzzle, I recommend you give it a try first.

The history has been unkind to its brightest minds: Galois, Galileo, Archimedes, and others met
tragic fates. Join me in unraveling a mysterious murder from two millennia ago![^warning].

![The death of Archemedis](https://math.nyu.edu/~crorres/Archimedes/Death/Degeorge/degeorge.png)

--------

Pre-compiles in Ethereum is fascinating. There is a rich-history behind introducing each of them, a
history that is full of drama[^blake]. Even though, we have nine of them today, people can't wait to
get more precompiles in EVM[^EIPS]. They are also a hot mess for security.[^modexp] They also share
some quirks that we'll explore soon.


Long time ago, one of the most requested feature in solidity was to skip the `extcodesize` check
that solidity performs for every high-level call. Such a check is important, because EVM assumes
that calls to empty addresses are successful by default. Whether or not this is a quirk or not is
upto debate, but that is another conversation.

This long requested feature in Solidity was meant to save gas (800 gas back then, before EIP-2929),
as in many cases the contract addresses were known in advance to have code. There were varying
proposals, from just dropping into assembly, to introducing new syntaxes to skip these
checks. Solidity settled for a solution that was in-between: the check was skipped if the function
had return values! See: [#12205](https://github.com/ethereum/solidity/pull/12205).

*Why did this work?* Because if a function returns data, Solidity would check if the
`returndatasize` (a cheap instruction) is at least as big as the data necessary for the returned
variable, before proceeding to decode the return variables. [^returndatasize]

If the `returndatasize()` is less than what it's supposed to be, then the function would immediately
revert. This was the case before and after the above PR #12205.

So you can skip the `extcodesize` check since the `returndatasize` check will revert anyway for empty
accounts. But is that really true? Unfortunately, a common theme in EVM is that there is an exception for
every rule. And in this case, there are accounts that have empty `extcodesize`, but can still return
data! These are precompiles!

So technically, the PR #12205, was a subtle breaking change in Solidity, but we concluded that this
is such an exceptional case that this breaking change is okay. Precompiles do not follow the ABI
standard, and therefore, calling them using a high-level call was idiosyncratic. If you want to see
the breaking change in action, you can see how the output of the test in
[#12219](https://github.com/ethereum/solidity/pull/12219/files) changed in the PR
[#12205](https://github.com/ethereum/solidity/pull/12205/files#diff-99e5627b1eba2ab69b09bafbc9d5001b7d7f899cf6d136477441715159b129c2).
This changed landed in Solidity version 0.8.10.

---------------

A peculiar precompile is the address 4. This is the identity precompile, which just returns back all
the data that's sent to it. This was originally designed for copying from memory (poor man's
`memcpy`), and `solc` at some point was using this for internal routines. However, various repricings
in the EVM lead to this copy routine being unreliable, and hence removed in later versions of the compiler.

----------------

There are several EIPs that enforce a certain magic return bytes to be returned in the correct case. This magic bytes is typically the selector of the function in question. However, identity
pre-compile combined with such standards create a very peculiar scenario where the magic checks can be made to succeed!

Consider the following interface `IMagicReturn`, that expects a magic 4-byte value to be returned in case of success.

```solidity
interface IMagicReturn {
    /// MUST return magic `foo.selector` in the valid case
    function foo() external returns (bytes4);
}

/// Function that enforces the magic check
function magicCheck(IMagicReturn magicReturn) {
    require(magicReturn.foo() == IMagicReturn.selector);
}
```

Since ABI encoding of an external call starts with the first the selector, the first 4 bytes
returned by identity precompile conceptually returns the correct magic byte.

This is still not enough to follow the spec, as Solidity expects at least 32-bytes returned in the
above case. But since `MagicReturn(address(0x4)).foo()` only returns 4-bytes, the high-level call
will revert.

But this can be easily be fixed by adding extra parameters to the function. Consider the interface below:

```solidity
interface MagicReturnWithExtraData {
    /// MUST return magic `foo.selector` in the valid case
    function foo(uint x) external returns (bytes4);
}

/// Function that enforces the magic check
function magicCheck(IMagicReturn magicReturn) {
    require(magicReturn.foo(type(uint).max) == IMagicReturn.selector);
}
```

In the above case, `MagicReturnWithExtraData(address(0x4)).foo(type(uint).max)` returns 36 bytes of
data. However, if you test this out, the call to `magicCheck ` will still revert!

Solidity adds too many safety checks for its own good. When you think you can get around a check, another
check will save the day. In the above case, there's an additional check that happens when decoding the `bytes4` return
type and checks if the remaining data in the word is `0`. In the above case, the `type(uint).max`
leaks into the same word and fails this check!

However, this check is only done by the ABI Encoder V2, and not V1. This is a key insight that gets used in the CTF, and this explains the curious `pragma abicoder v1;` in the CTF.

------

Now that we know how to use address 4 to satisfy the magic checks, let's now explore the idea of
using other pre-compiles to also satisfy such magic checks.

If you carefully look through the other [precompiled contracts](https://www.evm.codes/precompiled),
you can see that `sha256` can also be made to satisfy the magic check, provided that the inputs can
be mined to get a 4-byte match for the hash function's return value.

Unfortunately, the other hash-function `ripemd-160` at address 3, returns a zero padded data with
the first 12 bytes being 0 and the remaining being the hash. This means that you can only make this
pass a magic check if the selector for the function is `0`!

----

I wanted to dedicate the puzzle towards the legend of Hippasus's murder by Pythagoras. I asked
ChatGPT to write a theatrical version of this story, and prompted it to remove details until it
wasn't immediately obvious what the story was about. I confirmed this by copying over the story in a
different context and asking it questions about the story, especially around the killer and the
victim and it wasn't able to trace back to the original story. The goal of the CTF was to hide clues in the Solidity code that can be used to either prompt
chatGPT or just casually searching around about the historical reference.

The hidden clue was the formula:

$$3^2 + 4^2 = 5^2$$

which is a Pythagorean triplet, that satisfies the final math equation. Several people were really close to Pythagoras early on, but just never considered the idea that he could have been the killer! Great PR by the Pythagoreans.

The ABI encoder v1 was also another hidden clue that indicated that there is something phishy about
the return data decoding, indicating that the addresses you are supposed to deploy need not
necessarily be a regular address, hinting at precompiles being involved here.

The puzzle also required multiple things to work at the same time.

1. `pragma abi coder v1`.
2. At least a Solidity version `0.8.10`.


That's it, I gave you most of the details that'll allow you to solve the puzzle. You have about 3 days left to solve this in phase 2!

---

*The Murder of Hippasus by the Pythagoreans, by Midjourney*

![The Murder of Hippasus by Pythagoreans, by
Midjourney](https://user-images.githubusercontent.com/13174375/266861242-8ab79fc0-b58b-4375-9368-e748dd48ce24.png)



[^warning]: There are conflicting historical references to all the above stories. But I choose to go with the
    most entertaining stories.

[^EIPS]: https://github.com/ethereum/EIPs/pulls?q=is%3Apr+is%3Aclosed+precompile

[^blake]: Try talking to someone involved while `blake2f` was proposed as a precompile in EVM, and
    also on why `ripemd` even matters in EVM.

[^modexp]: https://twitter.com/_hrkrshnn/status/1600961228641546242

[^returndatasize]: This check is necessary before any doing any `returndatacopy`. This is because
    `returndatacopy` would revert on out-of-bounds access. I've spoken about this before (see my
    talk at DSS in Paris for example). The compiler generally avoids generating such code, even
    though the revert may be used to our advantage in some cases.

[^lowering]: This savings decreased to just 100 after changes introduced in EIP-2929.
