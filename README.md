## Formal verification of an automated market maker

An automated market maker is a program (a contract) that sets the price of a collection of cryptocurrency tokens. Investors give their assets to the contract, which are held in a "pool". When a user wishes to make a trade, the contract (1) computes an exchange rate, (2) receives the payment tokens (which are added to the appropriate pool) and (3) pays out the desired tokens (from a different pool). In exchange for keeping the pools sufficiently full, investors earn a bit of extra money, which is derived from the transaction fees collected by the contract.

In this blog post, we attempt formal verification of a simple [StarkNet](https://starkware.co/starknet/) automated market maker (AMM) using Horus, an open-soure CLI tool that uses SMT (satisfiability modulo theory) solvers to check user-defined program properties. The example contract is available on Github in a file called [`toy_amm_split.cairo`](https://github.com/NethermindEth/horus-checker/blob/master/tests/resources/golden/toy_amm/toy_amm_split.cairo). A working familarity with the StarkNet ecosystem is assumed in what follows.

We start with an excerpt picked precisely because it suggests that formal verification is useless. Consider the following subset of the `toy_amm_split.cairo` contract:

```cairo
%lang starknet
%builtins pedersen
from starkware.cairo.common.cairo_builtins import HashBuiltin


// A map from token type to the corresponding balance of the pool.
@storage_var
func pool_balance(token_type: felt) -> (balance: felt) {
}


// The maximum amount of each token that belongs to the AMM.

// Returns the pool's balance.
// @post $Return.balance == pool_balance(token_type)
func get_pool_token_balance{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(
    token_type: felt
) -> (balance: felt) {
    return pool_balance.read(token_type);
}
```

The function in question is `get_pool_token_balance()`. It is only a single line of code that reads from a storage variable called `pool_balance()`. This storage variable can be thought of as a global mapping from tokens to balances which can be read from and updated elsewhere in the contract. The comment beginning with `// @post` is a Horus annotation, a postcondition. In general, this is a boolean expression representing some property we desire our function satisfy in all cases **when the function has finished executing**.

In this example, it quite transparently says that the return value of the function, denoted `$Return.balance` (read: the element of the return tuple of `get_pool_token_balance()` named `balance`), is equal to the storage variable `pool_balance()` indexed at the value of the `token_type` argument. It is essentially a reimplementation of the function's logic. One could easily argue that if you made a mistake implementing the function itself, you are likely to make the same mistake implementation the specification (this is what we call the set of Horus annotations for a given function). This is known as the [_test oracle problem_](https://ieeexplore.ieee.org/document/7422146) in the literature, and as its name suggests, rears its head in both software testing and formal verification.

We can come up with a modified version of our example that demonstrates exactly this danger. Suppose instead we have two arguments to `get_pool_token_balance()`, one is a counter of some sorts, which we increment within the function body, and the other takes the place of `token_type: felt`. So we have:
```cairo
%lang starknet
%builtins pedersen
from starkware.cairo.common.cairo_builtins import HashBuiltin


// A map from token type to the corresponding balance of the pool.
@storage_var
func pool_balance(token_type: felt) -> (balance: felt) {
}


// The maximum amount of each token that belongs to the AMM.

// Returns the pool's balance and an incremented counter.
// @post $Return.z == x + 1
// @post $Return.w == pool_balance(y)
func get_pool_token_balance{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(
    x: felt, y: felt
) -> (z: felt, w: felt) {
    let (u) = pool_balance.read(y);
    return (z=x + 1, w=u);
}

func main{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}() -> (balance: felt, counter: felt) {
    let token_type = 15;
    let counter = 0;
    let (balance, counter) = get_pool_token_balance(token_type, counter);
    return (balance, counter);
}
```

There is now a `main()` function, and we call `get_pool_token_balance(token_type, counter)`. Note that the first parameter, `x`, is the one that gets incremented in `get_pool_token_balance()`, and we use the counter as a `token_type` when reading from the storage variable! So `main()` will certainly not work as we expect. But if we run this through Horus, we see:

```console
Warning: Horus is currently in the *alpha* stage. Please be aware of the
Warning: known issues: https://github.com/NethermindEth/horus-checker/issues

get_pool_token_balance
Verified

main [inlined]
Verified
```

So Horus cannot detect the bug. We've mixed up `x` and `y` in the function body, and we've mixed it up again in the `@post` conditions, and thus we have hit the _test oracle problem_. One could argue that the bug is really in `main()` and we've passed the wrong arguments to `get_pool_token_balance()`, but if we had many other functions in our program where the convention is always to pass the `counter` as the last argument, it would certainly be the case that the fault is in `get_pool_token_balance()`. Name mix-ups where the swapped variables have the same type are one of the hardest sorts of bugs to catch, and are made even more subtle when we choose bad, nondescriptive variable names, as in this version of `get_pool_token_balance()`.

We illustrate this shortcoming of formal verification methods (and software testing) in order to make clear the distinction between cases where using formal tools like Horus are a waste of time, and cases where they are nontrivially useful. One might come away from this discussion so far with the opinion that any sort of program verification is futile, and so let us look at an example of the latter case. Consider this modified version of our `get_pool_token_balance()` function:

```cairo
%lang starknet
%builtins pedersen
from starkware.cairo.common.cairo_builtins import HashBuiltin


// A map from token type to the corresponding balance of the pool.
@storage_var
func pool_balance(token_type: felt) -> (balance: felt) {
}


// The maximum amount of each token that belongs to the AMM.

// Returns the pool's balance.
// @post $Return.balance == pool_balance(h)
func get_pool_token_balance{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(
    a: felt,
    b: felt,
    c: felt,
    d: felt,
    e: felt,
    f: felt,
    g: felt,
    h: felt,
    i: felt,
    j: felt,
    k: felt,
) -> (balance: felt) {
    let (o) = pool_balance.read(h);
    let p = a + g + k - o + j + (2 * h);
    let q = p - e + c - f;
    let r = k + i + (5 * a) + f + k - q;
    let s = (4 * d) + g - r + (7 * e) - b;
    return (balance=-4 * a - b + c + 4 * d + 6 * e - 2 * f + 2 * g + 2 * h - i + j - k - s);
}
```

Notice that the specification for our function is functionally the same: `@post $Return.balance == pool_balance(h)`, just an assertion that we return the `pool_balance` at one of the function's arguments. But the logic by which we return this value is (contrivedly) opaque, and may harbor bugs. In a situation where we must perform several actions, make use of intermediate results, and return multiple values, the complexity may be infeasible to clear away while preserving the semantics of the function. This is where the usefulness of formal verification shines: in its ability to abstract-away a high-complexity implementation in order to guarantee the correctness of a low-complexity specification. Phrased as a rule of thumb, techniques like software testing and formal verification, in which you check write and check assertions about your program, are most appropriate when the assertions are simpler than the code. Here, when we say "simpler", we mean easier for the programmer to understand, since a bug in one's assertions renders verification methods useless. The probability of a bug in the assertions must be substantially lower than the probability of a bug in the implementation.

The function and specification above are an extreme, contrived example of this tenet: the postcondition is transparent, and the implementation is so unreadable as to be obfuscated. And indeed, Horus can make sense of this trivially where it would take a programmer some effort:

```
Warning: Horus is currently in the *alpha* stage. Please be aware of the
Warning: known issues: https://github.com/NethermindEth/horus-checker/issues

get_pool_token_balance
Verified
```
