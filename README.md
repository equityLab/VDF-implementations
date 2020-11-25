## Getting started
We provide VDF (Verifiable Delay Function) Implementations in Golang for two candidate VDFs which are the Piertzak VDF (Simple VDF) and Sloth (Modular Square Root). You can locate the candidates in
``
cd go_src/candidates
``.

We also provide a server providing VDF computation over HTTP. Run it by calling ``python server/main.py``. 

### Prerequisites

```
Python 2.7.1
```
Note: `vdf_interface` is built for MacOS. Rebuild VDF interface in go_src if you are operating on another OS.
### Run

In ``main.py`` we run an example VDF operation over a 128bit prime field.
```
python main.py
```
## Intro to VDF
VDF is a tuple of three algorithms:

<a href="https://www.codecogs.com/eqnedit.php?latex=Setup(\lambda,&space;T)\rightarrow&space;pp" target="_blank"><img src="https://latex.codecogs.com/gif.latex?Setup(\lambda,&space;T)\rightarrow&space;pp" title="Setup(\lambda, T)\rightarrow pp" /></a>

<a href="https://www.codecogs.com/eqnedit.php?latex=Eval(pp,x)&space;\rightarrow&space;(y,\pi)" target="_blank"><img src="https://latex.codecogs.com/gif.latex?Eval(pp,x)&space;\rightarrow&space;(y,\pi)" title="Eval(pp,x) \rightarrow (y,\pi)" /></a>

<a href="https://www.codecogs.com/eqnedit.php?latex=Verify(pp,x,y,\pi)&space;\rightarrow&space;\{accept,reject\}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?Verify(pp,x,y,\pi)&space;\rightarrow&space;\{accept,reject\}" title="Verify(pp,x,y,\pi) \rightarrow \{accept,reject\}" /></a>

### Modular Square Root

#### Evaluation
Sloth, or modular sqaure root, is one of the simplest yet effective candidate for verifiable delay function.

Given a prime field ``p``, starting value ``x``, and delay parameter ``t``:

We assume that finding ``x^2 % p`` is much easier than finding ``sqrt(x) % p``.
This is because the fastest existing algorithm to find modular square root of ``x``
is ``x^((p+1)/4) % p``, which requires a lot more operations than simply squaring ``x``.

This also implies that as the prime field ``p`` gets larger, more operations are required.

To implement Sloth, we first select a prime field ``p`` where ``p%4=3``. When we evaluate starting value ``x``, 
we have to first check whether ``x`` is a quadratic residue by checking whether 
``a^((p-1)/2)%p=1``. If not, we simply change starting value ``x`` to ``(-x)%p``.

#### Chaining

In Sloth, the ``t`` parameter specifies the number of iterations the VDF evaluation has undergone.
Since ``sqrt(x)->y``, we can extend the delay of Sloth by performing the same evaluation over ``y`` with 
``sqrt(x)-> y , sqrt(y)-> y_1 , sqrt(y_1)->y_2 ...``. This way long as the prime field (>256 bits)is large enough, 
the length of delay provided from adjusting ``t`` should be more than enough.

 

#### Proof Generation
Sloth is fairly straightforward when it comes to verification. We do not need to submit a proof 
for a basic Sloth implementation. Verifiers can simply square ``y`` once or ``y_t`` t+1 times to 
acquire the starting value ``x``.

For a faster verification, we can use SNARK proofs to be submitted along with ``y``. 
However, if we want better a verification vs. evaluation gap, there are better candidates 
for it. 

### Time Proof
In our protocol,``t`` is the time parameter for proof of elapsed time. In the context of Sloth, ``t`` 
is the number of modular square root evaluation. ``t`` can be computed incrementally for it to become a time proof.

##Implementation

This section is up for debate depending on the properties of Tendermint

After submitting the take order information adhering to 0x schema, the user will generate `x` value by hashing the signature 
provided in the order information. 
Using `x`, user begins to calculate VDF (Sloth for example): ``sqrt(x)-> y , sqrt(y)-> y_1 , sqrt(y_1)->y_2 ...`` until it reaches a 
satisfactory checkpoint `y_t`. User then submits `y_t, t , x , order_hash` to a relayer so that it can map them to the 
order information the user submitted previously. The relayer will then perform `vdf_verify` in `check_tx`
to ensure the validity of the user's VDF proof. In sloth's case, the relayer can simply calculate `((y_t)^2)^(t+1)==x`. 

After `check_tx`, the relayer will append `t` to the user's existing order information. Since traders are allowed to
submit multiple transactions for time proofs (ex: `tx_1: {y_t , t ,x} tx_2: {y_2t , 2t , x} tx_3: {y_3t , 3t , x}`),
the relayer should also check if the `t` is bigger than the existing `t` attached to the trader's information before
performing any VDF verification.

Before a block is mined, the relayers match take orders with make orders on a `t`-priority basis. 
Meaning that for each make order, the taker orders
are sorted by descending `t` and filled in that order.
