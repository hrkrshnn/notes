# Optimal way to shuffle a stack in EVM

Given a stack arrangement $a_{1}, \cdots, a_{17}$ what's the optimal way to transform it into
$b_{1}, \cdots, b_{17}$ where $b_i$ is a rearrangement of $a_i$ for $i \in {1, \cdots, 17}$.

If we assume that all values $a_1, \cdots, a_{17}$ are distinct and that we only want to perform
$\operatorname{swap}$ operations, there is a beautiful connection with [permutation
groups](https://en.wikipedia.org/wiki/Permutation_group), the problem can be converted into a known
group theory question.

Given the set $X = \{1, \cdots, 17\}$, the set of permutations of $X$ forms a group under composition
known as the permutation group, let's call it $S_{17}$. For the general case of $n$ elements, the
group is denoted by $S_n$. We'll start by introducing a notation to represent permutations. We'll
look at some general properties of the permutation group before applying them in the EVM context.

We define a $k$-cycle $(c_1\ c_2\ \cdots\ c_k)$ where $c_1, \cdots, c_k \in \{1, \cdots, n\}$ as the
permutation that performs the transformation $c_1 \mapsto c_2, c_2 \mapsto c_3, \cdots, c_{k-1} \mapsto c_k, c_k \mapsto c_1$.

As a concrete example, let's take the permutation in $S_4$ which translates $\{1, 2, 3, 4\}$ into
${2, 4, 3, 1}$. This permutation can be represented as $(1\ 2\ 4)$. It's clear that $(4\ 1\ 2)$ and
$(2\ 4\ 1)$ are identical representation of the same permutation. Therefore, as convention, we will
choose the representation where the first element is the smallest, i.e., $(1\ 2\ 4)$ in this case.

These cyclic representation is key to understanding permutations in general. This is due to the fact
that any permutation can be decomposed as a product of disjoint cycles. This is not 

## EVM context

In the EVM context, $\operatorname{swap}_i$ is the 2-cycle $(1\ (i + 1))$, e.g.,
$\operatorname{swap}_1$ is $(1\ 2)$ and $\operatorname{swap}_{16}$ is $(1\ 17)$. It's straightforward to see that
the permutations $(1\ 2), \cdots (1\ 17)$ can generate the group $S_{17}$.

Our optimality question can be translated as the following: given a permutation in $S_{17}$, what is
the smallest way to decompose it as a product of permutations $(1\ 2), (1\ 3), \cdots, (1\ 17)$?

Given a $k$-cycle $(a_1\ a_2\ \cdots a_k)$, how can we decompose this into 2-cycles of the form
$(1\ i)$ for $i \in \{1, \cdots, n\}$? 

Note that $(a_1\ a_2\ \cdots\ a_k) = (a_1\ a_k) (a_1\ a_{k - 1}) \cdots (a_1\ a_2)$.

Also notice that $(i\ j) = (1\ j)(1\ i)(1\ j) = (1\ i)(1\ j)(1\ i)$. In other words, two swapping
the $i$-th and $j$-th element in the stack is the same as
$\operatorname{swap}_i \cdot \operatorname{swap}_j \cdot \operatorname{swap}_i$, which is identical to
$\operatorname{swap}_j \cdot \operatorname{swap}_i \cdot \operatorname{swap}_j$.

Now we have all the steps needed to answer the original question: given a $k$-cycle,
$(a_1\ a_2\ \cdots a_k)$, this can be decomposed into
$(a_1\ a_k)\cdot (a_1\ a_{k-1}) \cdot (a_1 a_2)$, which can further be divided into 
$(1\ a_1)\cdot (1\ a_k)\cdot (1 \ a_1) \cdot (1 a_1) \cdot (1\ a_{k-1}) \cdot (1\ a_1) \cdots \cdot (1\ a_1) (1\ a_2) (1\ a_1)$

$$(1\ a_1)\cdot (1\ a_k) \cdot (1\ a_{k-1}) \cdots (1\ a_{k}) (1\ a_1)$$

That is, any $k$ cycle can be decomposed into at most $k + 2$ number of $\operatorname{swap}_i$
operation. But is this the optimal decomposition?



## Open questions
1. How does introducing the sequence $\operatorname{dup}_i$ followed by $\operatorname{swap}_i$-s
   and a $\operatorname{pop}$ change the optimality of the shuffling?
2. If the elements $a_i$ are not distinct, what's the optimal shuffling that only uses swaps? What
   if we can introduce $\operatorname{dup}$ and $\operatorname{pop}$?
3. How can we deal with the general case where variables can be repeated, generated on the fly and
   any EVM stack operation can be used? Can you improve the current heuristics in Solidity's stack
   shuffling algorithms? See [Daniel's talk](https://www.youtube.com/watch?v=RJQdycaEgIE0) and
   [slides](https://github.com/ethereum/solidity-summit/blob/master/2022/slides/1705_DanielKirchner_Generating%20EVM%20Bytecode%20from%20Yul%20in%20the%20New%20via-IR%20Pipeline.pdf)
   for some context as well as
   [StackLayoutGenerator.cpp](https://github.com/ethereum/solidity/blob/d5a78b18b3fd9e54b2839e9685127c6cdbddf614/libyul/backends/evm/StackLayoutGenerator.cpp#L145).

## Exercises
1. The widely used minimal proxy contract ([EIP-1167](https://eips.ethereum.org/EIPS/eip-1167)) is
   actually not minimal enough! In particular, it's stack arrangement can be improved. What's the
   ideal stack arrangement in this case?
2. Build a tool that generates the above optimal transformation.
3. The above ideas can be used to solve the [100 Prisoners
   problem](https://en.wikipedia.org/wiki/100_prisoners_problem). Read and solve this! There is a
   corresponding [YouTube video](https://www.youtube.com/watch?v=iSNsgj1OCLA0) as well.
