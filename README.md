# GSoC Report
</hr>

## Information
</hr>

Project: Arcilator Vectorization</br>
Organization: FOSSi Foundation</br>
Mentors: Fabian Schuiki, Martin Erhart</br>
Contributor: Mohamed Atef</br>

## Goals
</hr>

* Implement a pass to collect initial vectors.
* Implement simple canonicalization/optimization passes.
* Implement an initial cost model.
* implement splitting/merging/shuffling passes using the cost model.

**Note**: We talk much time in each of these goals, the project is so open-ended.</br>

### Implement a pass to collect initial vectors
</hr>
To get a good grasp of what this pass should do let's take a small example. given</br>
the following simple operations:</br>

```
a = b + c
d = e + f 
```
After we run our pass, this should be converted into</br>

```
[a, d] = [b, e] + [c, f]
```

#### Challenges
* The IR is sensitive to timing not the order of instructions (also a huge advantage).
* We need to find the initial vectors without introducing a cycle in IR.
* This is a chip, so every element communicates with the others somehow.

#### Approach
So in simple terms, we need to group isomorphic operations with no dependency between them.</br>
We started with the `arc.state` operation and gave each operation a rank if two operations<br>have the same rank then</br> 
this means they are independent so we put them in the same vector.</br>

#### Results:
We now have a full-working pass that can group the operations. After calculating</br>
operations we have after running `FindInitialVector` pass the number of operations is reduced</br>
by ~33%. I don't mean that we are 33% faster as the packing/unpacking and shuffling costs are</br>
very high, but that's an indication of a promising result</br>

#### Code
* Add Initial SLP support: https://github.com/llvm/circt/pull/7061
	* Fix related crashes: https://github.com/llvm/circt/pull/7100
* Modify VectorizeOp to accept any type: https://github.com/llvm/circt/pull/7087
* Add statistics to `FindInitialVectors` pass: https://github.com/llvm/circt/pull/7113

### Implement simple canonicalization/optimization passes
I didn't expect this step would take most of our work, but it turned out to be very important.</br>
As mentioned before packing/unpacking and shuffling costs are very high, we need to find</br>
ways to reduce them. how to do this? we need to pull as many operations as possible into the</br>
`VectorizeOp` body.

#### Challenges
One should research the IR to get one real-world canonicalization pattern.

#### Approach
We introduced **three** canonicalizations/optimizations:
* MergeVectorizeOps: A pass to merge two `arc.vectorize` operations if one is fully consumed</br>
	by the other.

* KeepOneVecOp: A pass to keep only one input vector if it is repeated multiple times. In reality</br>
one won't write the same multiple input vector many times, but after merging we can have repeated</br>
patterns.

* Add more cases to the MergeVectorizeOps pass to make it able to shuffle input vectors.

#### Results
* A merging pass and now we can have more ops in the `arc.vectorize` body. We also reduced the unpacking</br>
  cost of the used vector as we removed it and replaced it with the defining op operands</br>
```
// Before
%L:4 = arc.vectorize(%a, %b, %c), (%n, %p, %r): (i8, i8, i8, i8, i8, i8) -> (i8, i8, i8) {
  ^bb0(%arg0: i8, %arg1: i8):
    %ret = comb.and %arg0, %arg1: i8
    arc.vectorize.return %ret: i8
}
%C:4 = arc.vectorize(%L#0, %L#1, %L#2), (%o, %v, %q) : (i8, i8, i8, i8, i8, i8) -> (i8, i8, i8) {
  ^bb0(%arg0 : i8, %arg1: i8):
    %1692 = arc.call @Just_A_Dummy_Func(%arg0, %arg1) : (i8, i8) -> i8
    arc.vectorize.return %1692 : i8
}
```
```
// After
%C:4 = arc.vectorize(%a, %b, %c), (%n, %p, %r), (%o, %v, %q) : (i8, i8, i8, i8, i8, i8) -> (i8, i8, i8) {
  ^bb0(%arg0 : i8, %arg1: i8, %arg2: i8):
    %and = comb.and %arg0, %arg1: i8
    %call = arc.call @Just_A_Dummy_Func(%and, %arg2) : (i8, i8) -> i8
    arc.vectorize.return %call : i8
}
```
* After adding the `KeepOneVecOp` pass we have lower packing and unpacking costs, and removed the </br>
redundant input vectors</br>
**Example:** </br>
```
// Before
  %R:4 = arc.vectorize(%b, %e), (%b, %e) : (i8, i8, i8, i8) -> (i8, i8) {
    ^bb0(%arg0: i8, %arg1: i8):
      %ret = comb.mul %arg0, %arg1: i8
      arc.vectorize.return %ret: i8
  }
```
```
// After
%R:4 = arc.vectorize(%b, %e) : (i8, i8) -> (i8, i8) {
    ^bb0(%arg0: i8):
      %ret = comb.mul %arg0, %arg0: i8
      arc.vectorize.return %ret: i8
  }
```
* Shuffling the vectors before merging is a great optimization it lowered the vector shuffling costs</br> and opened more opportunities for merging. furthermore, it lowered removed the cost of vector unpacking</br> as now we removed the input</br>vector and merged its defining op body in the using op.</br>

#### Code
* Merge VectorizeOps: https://github.com/llvm/circt/pull/7146
* Keep one vector input if it's repeated: https://github.com/llvm/circt/pull/7146
	* Related crash: https://github.com/llvm/circt/pull/7429
* Shuffle Input vectors before merging: https://github.com/llvm/circt/pull/7394

### Implement an initial cost model
To see if the vectorization is profitable we need a way to calculate the cost of the module before</br> and after the vectorization so we introduced the `ArcCostModel`.

#### Challenges
* We need the pass to cache results and easy to extend e.g. add more operations in the future.
* We need approximate operation costs.

#### Result
* A great beta cost model.
* Simple Tester pass.

#### Code
* Arc Cost Model with Tester pass: https://github.com/llvm/circt/pull/7360

## Future work
</hr>

* Simple splitting pass: I am working on this.
* Fancy shuffling/splitting pass: One can research how to map these problems into linear optimization problems.

## OutComes
</hr>

When I started this project, I knew nothing about MLIR and Arcilator. I am still ignorant and have a long way to go,</br>
but I think I have a good understanding of MLIR and Arcilator. From the MLIR side, I think I can work independently,</br>
and for Arcilator, it's very challenging but I think It's clearer and I can work with easy/intermediate tasks on my own.</br>

The most important thing I think I got from the program was working with great minds in the arcilator community.</br>
I enjoyed the program a lot because of them. I am very thankful to Google and FOSSi for letting me know them.
