# Demo of Symbolic Fuzzing

implement symbolic execution (SE) on a given deep neural network (DNN). We can consider DNN as a specific type of programs and thus we can "execute" it. Symbolic execution would executes the DNN on symbolic inputs and returns symbolic outputs. In addition, we can represent the symbolic outputs as a logical formula and use a constraint solving such as `Z3` to check for "assertions", i.e., properties about the DNN.

## DNN
Consider the following DNN with 2 inputs, 2 hidden layers, and 2 outputs.
```python
# (weights, bias, use activation function relu or not)
n00 = ([1.0, -1.0], 0.0, True)
n01 = ([1.0, 1.0], 0.0, True)
hidden_layer0 = [n00,n01]

n10 = ([0.5, -0.2], 0.0, True)
n11 = ([-0.5, 0.1], 0.0, True)
hidden_layer1 = [n10, n11]

# don't use relu for outputs
o0 = ([1.0, -1.0], 0.0, False)  
o1 = ([-1.0, 1.0], 0.0, False)
output_layer = [o0, o1]

dnn = [hidden_layer0, hidden_layer1, output_layer]
```

After performing symbolic execution on dnn, we obtain symbolic states, represented by a logical formula relating inputs and outputs.

```python
symbolic_states = my_symbolic_execution(dnn)
...
"done, obtained symbolic states for DNN with 2 inputs, 4 hidden neurons, and 2 outputs in 0.1s"
assert z3.is_expr(symbolic_states)  #symbolic_states is a Z3 formula/expression

print(symbolic_states)
# And(n0_0 == If(i0 + -1*i1 <= 0, 0, i0 + -1*i1),
#     n0_1 == If(i0 + i1 <= 0, 0, i0 + i1),
#     n1_0 ==
#     If(1/2*n0_0 + -1/5*n0_1 <= 0, 0, 1/2*n0_0 + -1/5*n0_1),
#     n1_1 ==
#     If(-1/2*n0_0 + 1/10*n0_1 <= 0, 0, -1/2*n0_0 + 1/10*n0_1),
#     o0 == n1_0 + -1*n1_1,
#     o1 == -1*n1_0 + n1_1)
```

We can use a constraint solver such as `Z3` to query various things about this DNN from the obtained symbolic states:

Generating random inputs and obtain outputs
```python
z3.solve(symbolic_states)
[n0_1 = 15/2,
 o1 = 1/2,
 o0 = -1/2,
 i1 = 7/2,
 n1_1 = 1/2,
 n1_0 = 0,
 i0 = 4,
 n0_0 = 1/2]
```

Simultating a concrete execution
```python
 i0, i1, n0_0, n0_1, o0, o1 = z3.Reals("i0 i1 n0_0 n0_1 o0 o1")

 # finding outputs when inputs are fixed [i0 == 1, i1 == -1]
 g = z3.And([i0 == 1.0, i1 == -1.0])
 z3.solve(z3.And(symbolic_states, g))  # we get o0, o1 = 1, -1
 [n0_1 = 0,
 o1 = -1,
 o0 = 1,
 i1 = -1,
 n1_1 = 0,
 n1_0 = 1,
 i0 = 1,
 n0_0 = 2]
```

Checking assertions
```python
 print("Prove that if (n0_0 > 0.0 and n0_1 <= 0.0) then o0 > o1")
 g = z3.Implies(z3.And([n0_0 > 0.0, n0_1 <= 0.0]), o0 > o1)
 print(g)  # Implies(And(i0 - i1 > 0, i0 + i1 <= 0), o0 > o1)
 z3.prove(z3.Implies(symbolic_states, g))  # proved

 print("Prove that when (i0 - i1 > 0 and i0 + i1 <= 0), then o0 > o1")
 g = z3.Implies(z3.And([i0 - i1 > 0.0, i0 + i1 <= 0.0]), o0 > o1)
 print(g)  # Implies(And(i0 - i1 > 0, i0 + i1 <= 0), o0 > o1)
 z3.prove(z3.Implies(symbolic_states, g))
 # proved

 print("Disprove that when i0 - i1 >0, then o0 > o1")
 g = z3.Implies(i0 - i1 > 0.0, o0 > o1)
 print(g)  # Implies(And(i0 - i1 > 0, i0 + i1 <= 0), o0 > o1)
 z3.prove(z3.Implies(symbolic_states, g))
 # counterexample
 # [n0_1 = 15/2,
 # i1 = 7/2,
 # o0 = -1/2,
 # o1 = 1/2,
 # n1_0 = 0,
 # i0 = 4,
 # n1_1 = 1/2,
 # n0_0 = 1/2]
```



