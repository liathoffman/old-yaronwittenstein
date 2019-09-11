# svm

This blog post intends to give a high-level overview of [spacemesh][spacemesh] new [svm][svm] project and a brief overview of its future.
So...  <br>

What's _svm_?
_svm_ stands for Spacmesh Virtual Machine. It's based on the wonderful [wasmer][wasmer] WebAssembly runtime.
This article assumes basic knowledge about what's WebAssembly and terms like: `host, module, import object, instance` etc.

If you're not familiar with these, I highly recommend reading some of the links under _WebAssembly Articles_ at the end of this article.

_svm_ is designed for running smart-contracts under any spacemesh full-node.
There is a great buzz around using WebAssembly outside of the browser, and blockchain-related projects are a popular use-case.

Running smart-contracts transactions must produce the same output given the same starting point across each network node.
It should run deterministically regardless of the Operating-System and hardware used.

Using WebAssembly for smart-contract fits like a glove for this profound requirement (i.e portability) since while WebAssembly instructions abstraction are pretty low-level,
yet it's operating-system and hardware agnostic. Another key selling point is that WebAssembly programs can call the host (a.k.a VM/runtime)
for invoking functions (see: [WebAssembly Imports][WebAssembly Imports]).

Needless to say why security is so critical for running a smart-contract under a full-node.
WebAssembly programs run within a sandbox environment by design. Another big win for using WebAssembly.

_svm_ is essentially taking [wasmer][wasmer] which is a general-purpose WebAssembly runtime and extending and tailoring to suit spacemesh needs.

Actually, _svm_ could be used by other blockchain projects that would like to have WebAssembly-based smart-contracts.
It's a standalone project, and like every line of spacemesh code is 100% open-source:
https://github.com/spacemeshos/svm


Now, that we are all aligned with what _svm_ is about, let's talk about its components.

...


### Contract Storage

Each smart-contract has it's own internal persistent storage.
Without having disk persistence, we won't be able to run stateful programs.
Leaving us with feeble and trivial contracts solutions.

Since we're in the blockchain world, we need to be able to do sync starting from a given snapshot. Hence the contract storage is too, snapshot-oriented.
It means that when we do operations on it, we do them with an explicit context. We call that context/snapshot the `contract state`.
That state is derived from its underlying data, deterministically of course. More on that later.

The `contract storage` abstractions can be viewed hierarchically, when `kv` being the most low-level abstraction and `page slice cache` the most high-level
abstraction.

Let's dig a bit deeper...

#### Key-value store

A key-value map between a slice of bytes denoting the `key` to another slice of bytes, representing the `value`.
A key-value is stateless. It isn't aware of any `contract state`. Currently, _svm_ has `key-value` interfaces for `in-memory` (useful for tests), `leveldb` and `rocksdb`.
The `key-value store` serves as a companion for other `contract stoarge` components.

![svm-kv][svm-kv]

#### Pages Storage

In any standard strongly typed programming language, the global variables are known ahead of runtime, during compile-time.
Meaning, no new global variables can be added at runtime. The global variables live throughout the lifetime of the program.
Maybe there are workarounds in some languages but you get the point...

Similarly, in _svm_, we assume each smart-contract program is aware during its compile-time of its storage needs.
One program may require 5 variables of `32 bits` each (20 bytes in total), when other program uses 2 booleans (1 byte each)
and 4 integers of `64 bits` resulting in `2 + 4 * 8 = 34` bytes.

Now, from the `pages storage` point of view, there neither variables nor slices of bytes.
From `pages storage` point of view, there are only pages. Each page is of a constant size (_svm_ currently set `4096 bytes` as the page size).

If for example our page size is defined as `10 bytes`, then a program demanding `20 bytes` for its variables will consume 2 pages.
A program asking `32 bytes` for its variables will use 4 pages. The last page will contain 2 used bytes, and 8 unused zeroed bytes.

Within a smart contact program, each page is assigned a numeric index.
The page index is a sequential integer, local to the program, starting at 0.
The first page of any program will have `page index #0`, the 2nd page will have `page index #1` and so on.

The reason for that rough division it that we want to minimize disk access. Two adjacent program variables under the same page will
trigger loading only one page read operation.

Each program page has an associated hash, called the `page hash`.
The `page hash` is derived from the contract account address, its page index, and raw content.

![svm-pages][svm-pages]

#### Page Cache

The `page cache` is a caching layer on top of `pages storage`. If during a program run we'll ask to load a page more than once, then
only the first time will result in a disk hit (Operating System read system-call if to be more precise, since the Operating-System has its own
page cache, so we may not really hit the disk physically).

Once a running program issues a write-operation, we will mark the page as dirty. Committing the dirty pages
via the `page cache` takes place only after the smart contract finished its running. If the smart contract failed we discard the dirty pages.
The `page cache` interface isn't accessible to the smart contract code

The smart contract talks directly only to the `page slice cache` which we'll move next to explain.


#### Page Slice Cache

The `page slice cache` is the top-level abstraction in the hierarchy. _svm_ smart contracts talk directly only to the `page slice cache` (`page slice` for short).
We'll refer often to both `page slice` and `contract storage` interchangeably.

A running program trying to read a variable will translate to asking _svm_ for the corresponding page-slice, which is an array of bytes representing that variable.
A variable is an abstraction for running programs. There is no such thing as a variable for the `contract storage`, only `page slices`.

Each `page slice` has a sequential index, local to the program, starting at zero.
So the compiled program top variable is, `page slice #0`, the next variable is `page slice #1` and so on.
I say compiled program since a compiler may decide to reorder variables order as an optimization during the compilation phase.

If `slice #0` of a program represents a boolean and `slice #1` a `32 bit` integer, we say that `slice #0` is located under `page #0, offset=0, length=1`
and that `slice #1` resides at `page #1, offset=1, length=4`.

So _svm_ programs deal with `page slices`. When we want to read `slice #0 offset=0, length=1` for the first time, this request will translate to asking
the `page cache` for `page #0` which will result in a cache miss. Then, it'll delegate the page load request to the underlying `pages storage`.
The `pages storage` will infer the desired `page hash` given the current running `contract address` and `contract state` and proceed to ask `kv` store for the actual fetch.

When _svm_ overrides a `page slice`, it will first make sure it's been loaded, meaning the associated `page` to that `page slice` should be loaded.
Then we'll mark the `page slice` as dirty. When committing smart contract changes upon successful execution, we will ask the `page slice storage` to
commit only the dirty `page slices` which will translate to committing the associated dirty pages. Then we'll compute the new `page hash` for each committed dirty page
and calculate the new `contract state`, returning it as part of the `execution receipt`.


### SVM Registers

_svm_ registers are non-persistent ephemeral fixed-sized buffers used for alleviations of running smart-contracts.
Each _svm_ running program can take advantage of these buffers when otherwise it'd ask the runtime for more temporary memory cells,
or use the runtime stack. Each running _svm_ program is currently initialized with fixed registers settings of various sizes.

Additionally, using registers API can add some readability to the programs WebAssembly code.
As an example, let's assume our full-node is using `20 bytes (<=> 160 bits`) for account addresses and that a balance
is represented as `i64` (which is a WebAssembly primitive).

If we'd like to ask the full-node for a balance given its account address under a register, we might come up with runtime vmcall like `svm_get_balance_from_reg(160, 1) -> i64`
meaning: "read `register #1` of type `160 bits` for the desired account address and return its balance as an `i64` integer".

Currently, the svm registers settings for each running program consist of:
* 16 registers of `32 bits`
* 16 registers of `64 bits`
* 8 registers of `160 bits`
* 4 registers of `256 bits`
* 4 registers of `512 bits`

These numbers are of course subject to change between `svm` versions.

Another future initiative we may implement is adding to the contract program metadata the required registers settings required for its run.
Consequently, when we'll compile a high-level programming language targeted to the _svm_ flavored WebAssembly (see the `SMESH` section later), we'll decide as part
of the compilation process how many registers we'd like to have and how many of each type.

![svm-registers][svm-registers]

### Runtime API

On top of the `contract storage` and `contract registers` lies the high-level _svm_ Runtime API as seen from the outside world.
Basically, the runtime API should assist the full-node with the deployment of new contracts and executions of already deployed contracts.
The Runtime API is the gateway from which _svm_ can be seen as a black box.


#### Deployment

So a new smart-contract program has been born and it's being broadcast to the network as part of a `contract deployment` network transaction.
Hopefully, it'll be mined and get inserted into the chain (mesh in spacemesh case, but it's doesn't matter here).

Now, the actual contract deployment within a full-node involves:
* Storing the contract under what we call a `contract store`
* Creating a new contract account. Its address will be deterministically derived from the `contract deployment` transaction.
(`author account address`, `author account nonce`, `contract code` etc.)
* Initializing the `contract state` to `00...00` (all zeros).
* Setting the initial account balance if stated (defaults to zero).


Now we're ready to start executing `smart contract` transactions.

#### Execution

A smart contract execution transaction will contain:
* Contract address
* Transaction sender address
* Function name
* Function arguments
* Other


<br/>

The Runtime will:
* Load the contract WebAssembly code from the `contract store`
* Compile it to native code, called a `WebAssembly module` thank you [wasmer][wasmer]!
* Create a WebAssembly `import object` with all _svm_ built-in vmcalls (storage/register/full-node)

The `import object` will be initialized with the `contract address` and `contract state` provided by the full-node.

* Instantiate a new _svm_ instance, meaning instantiate a [wasmer][wasmer] instance, with the crafted `import object`.
* Call the desired function with its given arguments.
* Return an `execution receipt`. For now, it will only say whether the program execution succeeded or failed.

If the `execution receipt` reported a success, we'll commit all the storage changes to the `contract storage` and compute the new `contract state`.
Otherwise, we will discard the storage pending changes and remain with the same `contract state`.


### Runtime C-API & Golang binding

[wasmer][wasmer] and hence _svm_ are written in Rust.
Thankfully Rust has FFI bindings to C ABI. Allowing Rust to call C and vice versa.
We care about the C calling Rust direction. [wasmer][wasmer] comes with its own C-API out of the box.
_svm_ has it's own FFI API exposed (using the `wasmer c-api` internally).

spacemesh full-node is written in Golang which can interface with C ABI code using `cgo`.
So in order to integrate _svm_ with spacemesh full node, we should introduce a glue layer of spacemesh full-node `cgo`
to _svm_ Runtime C-API.

Luckily, [wasmer][wasmer] has a Golang client, [go-ext-wasm][go-ext-wasm] and there is now a [go-ext-wasm spacemesh fork][go-ext-wasm spacemesh]
that adds that glue layer for _svm_ needs.

...

### Roadmap

The first _svm_ milestone is now in the stage of being stabilized. And it's time to talk about will come after that.

So here's a brief about the future of _svm_:

#### Storage Unbounded Data-Structures

The current _svm_ only supports fixed-sized storage variables. There is no support for unbounded data-structures.
We'd like to add gradually support for these data-structures:
* list
* map
* set
* sorted-set

Each unbounded data-structure will manage its own state. The root `contract storage` will store these data-structures `state` in the same way
it stores booleans/integers. It's like having a reference type variable. Since a reference/data-structure state will be of a fixed-size, it can work.

Having these unbounded data-structures adds complexity to the `contract storage` which will have to additionally track changes of each unbounded data-structure.
Committing `contract storage` changes will now first require computing each unbounded data-structure new state, and updating the `contract storage`
matching `page slice`. Only after that, we could do a batch commit of all the storage changes at one shot.

Actually, this is a recursive process since an unbounded data-structure may hold unbounded data-structures too.
Say, we have a list of `map`s, or a `map` from `string` to `list`. It means we'll need to traverse first the most internal unbounded data-structures
and then, each will notify its parent about its new state and so on until reaching the top level `contract storage` (what we have today).

We need to do more research on how to represent each unbounded data-structure. We may end up having a [MPT][MPT] Ethereum-style for each
such data-structure. Maybe different data-structures will have different methods for managing their state.


#### Gas Metering

This is, of course, goes without saying. We can't let a program run forever due to the [halting problem][halting problem].
We are still thinking of how to make the gas metering more elegant when possible. More on that in the future.


#### Contract to Contract Calls

We'll want to have an explicit distinction between a contract calling other contract and a using another library/package/dependency.
As long a program relies on libraries, it won't issue a call to other accounts.
In the rest of the cases, we do want to option to explicitly call another contract, resulting in a spawn of a new _svm_ instance.
More about the using dependencies under `Code Reuse`.


#### Structured Events with expiration

We want to be able to add structured events for easier searching and we'd like to set an expiration on them in order to save disk space.
In case a program desires permanent events, the solution will be to use a general `contract storage` variable suited.
(see `Storage Unbounded Data-Structures` above).


#### Code Reuse

One of the big must-have for _svm_ runtime is the ability to let programs reuse code written by other people.
We'd like _svm_ programs to be able to use other packages of code. _svm_ smart contracts should be able to call not only runtime builtin vmcalls,
but also other packages loaded upon instantiation. Part of these packages may be written in any programming language compiled to WebAssembly.
(see also `SMESH` later).

These packages may not be associated with other contracts accounts (not sure how it'll be represented yet), but they still must be stored on-chain.
It's like having access to `RubyGems/npm/crates.io/other packages-managers` accessible on the chain. (documentation or debugging info will be stored off-chain).

I know there is an early-stage work being done on WebAssembly modules interoperability. Lin Clark has published a few weeks ago a great article
about [WebAssembly Interface Types][WebAssembly Interface Types]. This seems a big step towards that direction.

I could imagine feeding a wasmer module with its dependencies modules and compile it natively under one executable unit of code.
Or maybe instead of having one compiled module. We could tell wasmer when we build the `import object`, that we're interested not only on the
predefined runtime vmcalls, but also of functions that exist in other WebAssembly modules, serving us as dependencies for us.
In such a case we could look at programs as importing runtime dynamic vmcalls.

This idea is somewhat similar to having an operation-system pre-compiled with built-ins (like system-calls) vs dynamically loading kernel modules at runtime.


#### SMESH

The puzzle will never be complete without having high-level friendly programming-language that will compile to _svm_ WebAssembly code.
(i.e WebAssembly using the _svm_ runtime vmcalls). There has been done initial planning for this future programming-language under the name _SMESH_.
The actual work on _SMESH_ will probably start before reaching all the _svm_ milestones described above.
I've given a talk about the motivation for having _SMESH_ [here][Spacemesh smart contracts research].

...

### Summary

In this article, we've reviewed the work being done so far for _svm_ - Spacemesh Virtual Machine, and the motivation behind it.
Then, we've talked about the next steps for _svm_ and mentioned _SMESH_, the future high-level programming-language that will compile to _svm_ WebAssembly code.

In order to fulfill these ambitious goals, we need to add new people :wink:

So if you're a Rust developer interested in compilation and programming-languages please don't shy away...

Here is a link to [spacemesh contact page][spacemesh contact page]
or you can send me directly an email to _yaron.wittensten@gmail.com_

If you're interested in contributing to _svm_, please take a look at the [svm issues][svm issues] page.
Please contact us before starting doing any work, so that we'll both be on the same page.

We'll also provide [gitcoin][spacemesh gitcoin] bounties :moneybag::moneybag: for contributors.


References:

* [spacemesh][spacemesh]
* [svm][svm]
* [wasmer][wasmer]
* [go-ext-wasm spacemesh][go-ext-wasm spacemesh]
* [Spacemesh smart contracts research][Spacemesh smart contracts research]
* [spacemesh contact page][spacemesh contact page]


WebAssembly Articles:

* [WebAssembly Official Page](https://webassembly.org/)
* [Introduction to WebAssembly](https://rsms.me/wasm-intro)
* [WebAssembly: How and why](https://blog.logrocket.com/webassembly-how-and-why-559b7f96cd71/)
* [Creating and working with WebAssembly modules](https://hacks.mozilla.org/2017/02/creating-and-working-with-webassembly-modules/)



[svm]: https://github.com/spacemeshos/svm
[spacemesh]: http://spacemesh.io
[spacemesh gitcoin]: https://gitcoin.co/profile/spacemesh
[wasmer]: https://wasmer.io
[wasmer go client]: https://github.com/spacemeshos/go-ext-wasm
[Spacemesh smart contracts research]: https://www.youtube.com/watch?v=mcvBXQ0SWJM
[WebAssembly Imports]: https://webassembly.org/docs/modules/#imports
[WebAssembly Interface Types]: https://hacks.mozilla.org/2019/08/webassembly-interface-types/
[RocksDB]: https://rocksdb.org/
[go-ext-wasm]: https://github.com/wasmerio/go-ext-wasm
[go-ext-wasm spacemesh]: https://github.com/spacemeshos/go-ext-wasm
[MPT]: https://medium.com/codechain/modified-merkle-patricia-trie-how-ethereum-saves-a-state-e6d7555078dd
[halting problem]: https://en.wikipedia.org/wiki/Halting_problem
[spacemesh contact page]: https://spacemesh.io/contact/
[svm issues]: https://github.com/spacemeshos/svm/issues

[svm-kv]: ./../images/svm-kv.png
[svm-contract]: ./../images/svm-contract.png
[svm-pages]: ./../images/svm-pages.png
[svm-registers]: ./../images/svm-registers.png
