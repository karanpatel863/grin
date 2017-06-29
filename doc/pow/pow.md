Grin's Proof-of-Work
====================

[WIP and subject to review, may still contain errors]

This document is meant to outline, at a level suitable for someone without prior knowledge, 
the algorithms and processes currently involved in Grin's Proof-of-Work system. We'll start
with a general overview of cycles in a graph and the Cuckoo Cycle algorithm which forms the 
basis of Grin's proof-of-work. We'll then move on to Grin-specific details, which will outline
the other systems that combine with Cuckoo Cycles to form the entirety of mining in Grin.

Please note that Grin is currently under active development, and any and all of this is subject to
(and will) change before a general release.

# Graphs and Cuckoo Cycles

Grin's basic Proof-of-Work algorithm is called Cuckoo Cycle, which is specifically designed
to be resistant to Bitcoin style hardware arms-races. It is primarily a memory bound algorithm,
which, (at least in theory,) means that solution time is limited to the speed of a system's RAM
rather than processor or GPU speed. As such, mining Cuckoo Cycle solutions should be viable on
most commodity hardware, and require far less energy than most other GPU, CPU or ASIC-bound 
proof of work algorithms.

The Cuckoo Cycle POW is the work of John Tromp, and the most up-to-date documentation and implementations
can be found in [his github repository](https://github.com/tromp/cuckoo). The
[white paper](https://github.com/tromp/cuckoo/blob/master/doc/cuckoo.pdf) is the best source of
further technical details. 

## Cycles in a Graph

Cuckoo Cycles is an algorithm meant to detect cycles in a random bipartite graphs graph of N nodes and M edges.
In plainer terms, a Node is simply an element storing a value, an Edge is a line connecting two nodes,
and a graph is bipartite when it's split into two groupings. The simple
graph below, with values placed at random, denotes just such a graph, with 8 Nodes storing 8 values 
divided into 2 groups (one row on top and one row on the bottom,) and zero Edges (i.e. no lines
connecting any nodes.) 

![alt text](images/cuckoo_base_numbered_minimal.png)

*A graph of 8 Nodes with Zero Edges*

Let's throw a few Edges into the graph now, randomly:

![alt text](images/cuckoo_base_numbered_few_edges.png)

*8 Nodes with 4 Edges*

We now have a randomly-generated graph with 8 nodes (N) and 4 edges (M), or an NxM graph where 
N=8 and M=4. Our basic Proof-of-Work is now concerned with finding 'cycles' of a certain length 
within this random graph, or, put simply, a path of connected nodes. So, if we were looking
for a cycle of length 3 (a path connecting 3 nodes), one can be detected in this graph, 
i.e. the path running from 5 to 6 to 3:

![alt text](images/cuckoo_base_numbered_few_edges_cycle.png)

*Cycle found*

Adjusting the number of Edges M relative to the number of Nodes N changes the difficulty of the 
cycle-finding problem, and the probability that a cycle exists in the current graph. For instance,
if our POW problem were to find a cycle of length 5 in the graph, the current difficulty of 5/8 (M/N)
would mean that all 4 edges would need to be randomly generated in a perfect cycle in order for
there to be a solution. If you increase the number of edges relative to the number of nodes,
you increase the probability that a solution exists:

![alt text](images/cuckoo_base_numbered_many_edges.png)

*MxN = 9x8  - Cycle of length 5 found*

So modifying the ratio M/N changes the number of expected occurrences of a cycle within a randomly
generated graph. 

For a small graph such as the one above, determining whether a cycle of a certain length exists is trivial. 
But as the graphs get larger, detecting such cycles becomes more difficult. For instance, does this
 graph have a cycle of length 7, i.e. 7 directly connected nodes?

![alt text](images/cuckoo_base_numbered_many.png)

*Meat-space Cycle Detection exercise*

The answer is left as an exercise to the reader, but the overall takeaway is that detecting such cycles becomes
a more difficult exercise as the size of a graph grows. It also becomes easier as M/N becomes larger, i.e. you add more edges relative to the number of nodes in a graph. 

## Cuckoo Cycles

The Cuckoo Cycles algorithm is a specialised algorithm designed to solve exactly this problem, and it does
so by inserting values into a structure called a 'Cuckoo Hashtable' according to a hash which maps nodes
into possible locations in two separate arrays. This document won't go into detail on the base algorithm, as
it's outlined in plain enough detail in section 5 of the 
[white paper](https://github.com/tromp/cuckoo/blob/master/doc/cuckoo.pdf). There are also several
variants on the algorithm that make various speed/memory tradeoffs, again beyond the scope of this document. 
However, there are a few details following from the above that we need to keep in mind before going on to more 
technical details of Grin's proof-of-work.
 
* The 'random' graphs, as detailed above, are not actually random but are generated by putting nodes through a
seeded hashing function, SIPHASH, generating two potential locations (one in each array) for each node in the graph.
The seed will come from a hash of a block header, outlined further below. 
* The 'Proof' created by this algorithm is a set of nonces that generate the cycle, which can be trivially validated by other nodes.
* Two main parameters, as explained above, are passed into the Cuckoo Cycle algorithm that affect the probability of a solution, and the
time it takes to search the graph for a solution: 
    * The M/N ratio outlined above, which controls the number of edges relative to the size of the graph
    * The size of the graph itself

How these parameters interact in practice is looked at in more [detail below](#mining-loop-difficulty-control-and-timing).

Now, (hopefully) armed with a basic understanding of what the Cuckoo Cycle algorithm is intended to do, as well as the parameters that affect how difficult it is to find a solution, we move on to the other portions of Grin's POW system.

# Mining in Grin

The Cuckoo Cycle outlined above forms the basis of Grin's mining process, however Grin uses Cuckoo Cycles in tandem with several other systems to create a Proof-of-Work.

### Additional Difficulty Control

In order to provide additional difficulty control in a manner that meets the needs of a network with constantly evolving hashpower
availability, a further Hashcash-based difficulty check is applied to potential solution sets as follows: 

If the SHA256 hash
of a potential set of solution nonces (currently an array of 42 u32s representing the cycle nonces,) 
is less than an evolving difficulty target T, then the solution is considered valid. More precisely, 
the proof difficulty is calculated as the maximum target hash (2^256) divided by the current hash, 
rounded to give an integer. If this integer is larger than the evolving network difficulty, the POW
is considered valid and the block is submit to the chain for validation.

In other words, a potential proof, as well as containting a valid Cuckoo Cycle, also needs to hash to a value higher than the target difficulty. This difficulty is derived from:

### Evolving Network Difficulty

The difficulty target is intended to evolve according to the available network hashpower, with the goal of
keeping the average block solution time within range of a target (currently 60 seconds, though this is subject
to change). 

The difficulty calculation is based on both Digishield and GravityWave family of difficulty computation, 
coming to something very close to Zcash. The refence difficulty is an average of the difficulty over a window of
23 blocks (the current consensus value). The corresponding timespan is calculated by using the difference between 
the median timestamps at the beginning and the end of the window. If the timespan is higher or lower than a certain
range, (adjusted with a dampening factor to allow for normal variation,) then the difficulty is raised or lowered
to a value aiming for the target block solve time.

### The Mining Loop

All of these systems are put together in the mining loop, which attempts to create 
valid Proofs-of-Work to create the latest block in the chain. The following is an outline of what the main mining loop does during a single iteration:

* Get the latest chain state and build a block on top of it, which includes
    * A Block Header with new values particular to this mining attempt, which are:
    
        * The latest target difficulty as selected by the [evolving network difficulty](#evolving-network-difficulty) algorithm
        * A set of transactions available for validation selected from the transaction pool
        * A coinbase transaction (which we're hoping to give to ourselves)
        * The current timestamp
        * A randomly generated nonce to add further randomness to the header's hash
        * The merkle root of the UTXO set and fees (not yet implemented)

        * Then, a sub-loop runs for a set amount of time, currently configured at 2 seconds, where the following happens:

            * The new block header is hashed to create a hash value
            * The cuckoo graph generator is initialised, which accepts as parameters:
                * The hash of the potential block header, which is to be used as the key to a SIPHASH function
                that will generate pairs of locations for each node in the graph. 
                * The size of the graph (a consensus value).
                * An easiness value, (a consensus value) representing the M/N ratio described above denoting the probability
                of a solution appearing in the graph
            * The Cuckoo Cycle detection algorithm tries to find a solution (i.e. a cycle of length 42) within the generated
            graph. 
            * If a cycle is found, a SHA256 hash of the proof is created and is compared to the current target
            difficulty, as outlined in [Additional Difficulty Control](#additional-difficulty-control) above.
            * If the SHA256 Hash difficulty is greater than or equal to the target difficulty, the block is sent to the
            transaction pool, propogated amongst peers for validation, and work begins on the next block.
            * If the SHA256 Hash difficulty is less than the target difficulty, the proof is thrown out and the timed loop continues.
            * If no solution is found, increment the nonce in the header by 1, and update the header's timestamp so the next iteration
            hashes a different value for seeding the next loop's graph generation step.
            * If the loop times out with no solution found, start over again from the top, collecting new transactions and creating
            a new block altogether.

### Mining Loop Difficulty Control and Timing

Controlling the overall difficulty of the mining loop requires finding a balance between the three values outlined above:

* Graph size (currently represented as a bit-shift value n representing a size of 2^n nodes, consensus value
  DEFAULT_SIZESHIFT). Smaller graphs can be exhaustively searched more quickly, but will also have fewer 
  solutions for a given easiness value. A very small graph needs a higher easiness value to have the same 
  chance to have a solution as a larger graph with a lower easiness value.
* The 'Easiness' consensus value, or the M/N ratio of the graph expressed as a percentage. The higher this value, the more likely
  it is a generated graph will contain a solution. In tandem with the above, the larger the graph, the more solutions 
  it will contain for a given easiness value.
* The evolving network difficulty hash.  

These values need to be carefully tweaked in order for the mining algorithm to find the right balance between the 
cuckoo graph size and the evolving difficulty. The POW needs to remain mostly Cuckoo Cycle based, but still allow for
reasonably short block times that allow new transactions to be quickly processed.

If the graph size is too low and the easiness too high, for instance, then many cuckoo cycle solutions can easily be 
found for a given block, and the POW will start to favour those who can hash faster, precisely what Cuckoo Cycles is 
trying to avoid. If the graph is too large and easiness too low, however, then it can potentially take any solver a 
long time to find a solution in a single graph, well outside a window in which you'd like to stop to collect new
transactions.

These values are currently set to 2^12 for the graph size and 50% for the easiness value, however they are only
temporary values for testing. The current miner implementation is very unoptimised, and the graph size will need
to be changed as faster and more optimised Cuckoo Cycle algorithms are put in place.

### Pooling Capability

[More detail needed here] Note that contrary to some concerns about the ability to effectively pool Cuckoo Cycle mining, pooling Grin's POW
as outlined above is relatively straightforward. Members of the pool are able to prove they're working on a solution
by submitting valid proofs that simply fall under the current network target difficulty.






