# Intermediate MP-SPDZ

If you want to learn how to write your first program in MP-SPDZ, read the [Hello World tutorial](https://openmpcc.gitbook.io/openmpcc/tutorials/helloworld).

In this tutorial, we summarize the features of the MP-SPDZ programming language and finish with a more advanced example computing a single-layer neural network.

## 1. Language Syntax Basics

MP-SPDZ syntax is Python-like. You can use:

### Variables and Types
- `sint`: secret-shared integer
- `sfix`: secret-shared fixed-point
- `regint`: public integer

```python
x = sint(5)           # secret-shared constant
y = regint(10)        # public constant
z = sfix(3.14)        # secret-shared fixed-point
```

---

### Arithmetic
You can use standard operators:

```python
a = sint.get_input_from(0)
b = sint.get_input_from(1)

c = a * b + 2
d = c % 5
print_ln("Result = %s", d.reveal())
```

---

### Conditionals
Branching must be **data-independent** in MPC.  
Use **if on public values**, or the ternary-like syntax for secrets.

```python
# Public branching
x = regint(5)
if x < 10:
    print_ln("x is less than 10")

# Secret branching
a = sint.get_input_from(0)
b = sint.get_input_from(1)
m = a < b     # secret comparison
c = m.if_else(a, b)   # choose a if m else b
print_ln("Minimum is %s", c.reveal())
```

---

### Loops and Control Flow
Loops must have a public bound.

```python
total = sint(0)
for i in range(5):
    x = sint.get_input_from(0)
    total += x

print_ln("Sum of 5 inputs = %s", total.reveal())
```

---

### Arrays and Vectors

```python
n = 3
arr = Array(n, sint)

# Input values
for i in range(n):
    arr[i] = sint.get_input_from(0)

# Sum array
s = sint(0)
for i in range(n):
    s += arr[i]

print_ln("Array sum = %s", s.reveal())
```

---

### Functions

You can define reusable functions with secret types.

```python
def secure_dot(x, y, n):
    result = sint(0)
    for i in range(n):
        result += x[i] * y[i]
    return result

# Example use
n = 3
vec1 = Array(n, sint)
vec2 = Array(n, sint)

for i in range(n):
    vec1[i] = sint.get_input_from(0)
    vec2[i] = sint.get_input_from(1)

dot = secure_dot(vec1, vec2, n)
print_ln("Dot product = %s", dot.reveal())
```

---

### Fixed-Point Arithmetic

MP-SPDZ supports **secret-shared fixed-point numbers** using `sfix`.

```python
a = sfix.get_input_from(0)
b = sfix.get_input_from(1)

c = a / b
print_ln("Division result = %s", c.reveal())
```

---

## 2. Putting It Together: Secure Linear Regression

```python
# Inputs: (x, y) pairs from party 0
n = 5
xs = Array(n, sfix)
ys = Array(n, sfix)

for i in range(n):
    xs[i] = sfix.get_input_from(0)
    ys[i] = sfix.get_input_from(0)

# Compute slope (beta) = Cov(x,y) / Var(x)
mean_x = sum(xs) / n
mean_y = sum(ys) / n

num = sfix(0)
den = sfix(0)
for i in range(n):
    num += (xs[i] - mean_x) * (ys[i] - mean_y)
    den += (xs[i] - mean_x) ** 2

beta = num / den
alpha = mean_y - beta * mean_x

print_ln("Regression line: y = %s * x + %s", beta.reveal(), alpha.reveal())
```

We will execute this simple linear regression with the dataset below:

```scss
1 2
2 3
3 5
4 4
5 6
```

The results should be `y = 0.9x + 1.3` when the program is executed.
Different protocols are available depending on your adversary/security model:
- `./semi2k-party.x` – semi-honest replicated sharing
- `./shamir-party.x` – Shamir secret sharing
- `./he-party.x` – homomorphic encryption-based
- `./malicious-shamir-party.x` – malicious security

Just replace the binary when running:

```
/Scripts/compile-run.py linear-regression.mpc mascot
Default bit length for compilation: 64
Default security parameter for compilation: 40
Compiling file linear-regression.mpc
WARNING: Probabilistic truncation leaks some information, see https://eprint.iacr.org/2024/1127 for discussion. Use 'sfix.round_nearest = True' to deactivate this for fixed-point operations.
Writing to Programs/Bytecode/linear-regression-FPDiv(2)_31_16-1.bc
Writing to Programs/Bytecode/linear-regression-TruncPr(10)_47_16-3.bc
Writing to Programs/Bytecode/linear-regression-FPDiv(1)_31_16-5.bc
Writing to Programs/Bytecode/linear-regression-TruncPr(1)_47_16-7.bc
Writing to Programs/Schedules/linear-regression.sch
Writing to Programs/Bytecode/linear-regression-0.bc
Hash: b4ce4c91a5243b9eb4b497f4a3cc8ec02147b5ccb4f8453ff567b7221299fa21
Program requires at most:
          10 integer inputs from player 0
        3054 integer bits
          34 integer opens
         282 integer triples
          46 virtual machine rounds
Compilation finished, running program...
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 0 linear-regression -pn 13629 -h localhost -N 2
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 1 linear-regression -pn 13629 -h localhost -N 2
Using statistical security parameter 40
Regression line: y = 0.899994 * x + 1.3
The following benchmarks are including preprocessing (offline phase).
Time = 1.51008 seconds 
Data sent = 85.65 MB in ~287 rounds (party 0 only; use '-v' for more details)
Global data sent = 150.818 MB (all parties)
```

## 3. Advanced Example: Secure Neural Network Inference

We’ll build a **1-hidden-layer feedforward network**:

- Input: vector of size `d`  
- Hidden layer: `h` neurons with activation  
- Output: one neuron with logistic activation  

Mathematically:

```
y = σ(W2 · φ(W1 x + b1) + b2)
```

where:
- `x`: input vector  
- `W1, W2`: weight matrices  
- `b1, b2`: biases  
- `φ`: activation (ReLU approximation)  
- `σ`: logistic sigmoid approximation  

---

### Step 1: Define Parameters

```python
# Parameters
d = 3   # input dimension
h = 4   # hidden layer size

# Input vector from party 1
x = Vector(d, sfix)
for i in range(d):
    x[i] = sfix.get_input_from(1)

# Weights and biases from party 0
W1 = Matrix(h, d, sfix)
b1 = Array(h, sfix)

for i in range(h):
    for j in range(d):
        W1[i][j] = sfix.get_input_from(0)
    b1[i] = sfix.get_input_from(0)

W2 = Array(h, sfix)
b2 = sfix.get_input_from(0)
for j in range(h):
    W2[j] = sfix.get_input_from(0)
```

---

### Step 2: Activation Functions

```python
def relu(x):
    # Approximate ReLU: max(x,0)
    return (x > 0).if_else(x, sfix(0))

def logistic(x):
    # Polynomial approximation of sigmoid
    return sfix(0.5) + x / (sfix(4) + x.abs())
```

---

### Step 3: Forward Pass

```python
# Hidden layer
hidden = Array(h, sfix)
for i in range(h):
    hidden[i] = W1[i].dot(x) + b1[i]
    hidden[i] = relu(hidden[i])

# Output layer
out = W2.dot(hidden) + b2
out = logistic(out)
```

---

### Step 4: Reveal Result

```python
print_ln("Neural network output = %s", out.reveal())
```

---

## 3. Full Program: `mini_nn.mpc`

```python
d = 3   # input dimension
h = 4   # hidden size

# Input vector from party 1
x = Vector(d, sfix)
for i in range(d):
    x[i] = sfix.get_input_from(1)

# Parameters from party 0
W1 = Matrix(h, d, sfix)
b1 = Array(h, sfix)
for i in range(h):
    for j in range(d):
        W1[i][j] = sfix.get_input_from(0)
    b1[i] = sfix.get_input_from(0)

W2 = Array(h, sfix)
for j in range(h):
    W2[j] = sfix.get_input_from(0)
b2 = sfix.get_input_from(0)

# Activation functions
def relu(x):
    return (x > 0).if_else(x, sfix(0))

def logistic(x):
    return sfix(0.5) + x / (sfix(4) + x.abs())

# Forward pass
hidden = Array(h, sfix)
for i in range(h):
    hidden[i] = relu(W1[i].dot(x) + b1[i])

out = logistic(W2.dot(hidden) + b2)

# Reveal result
print_ln("Neural network output = %s", out.reveal())
```
We will execute with the inputs below given as `Player-Data/Input-P0-0`:

```scss
0.1 0.7 0
0.3 0.3 0
0.4 0.5 0
0.6 0.2 1
0.8 0.6 1
0.9 0.3 1
0.2 0.8 0
0.5 0.4 1
0.7 0.9 1
0.3 0.9 0
```

The results can be found below.
```
./Scripts/compile-run.py mascot mini_nn.mpc 
Default bit length for compilation: 64
Default security parameter for compilation: 40
Compiling file mini_nn.mpc
WARNING: Probabilistic truncation leaks some information, see https://eprint.iacr.org/2024/1127 for discussion. Use 'sfix.round_nearest = True' to deactivate this for fixed-point operations.
Writing to Programs/Bytecode/mini_nn-TruncPr(4)_47_16-1.bc
Writing to Programs/Bytecode/mini_nn-LTZ(4)_32-3.bc
Writing to Programs/Bytecode/mini_nn-TruncPr(1)_47_16-5.bc
Writing to Programs/Bytecode/mini_nn-LTZ(1)_32-6.bc
Writing to Programs/Bytecode/mini_nn-FPDiv(1)_31_16-7.bc
Writing to Programs/Schedules/mini_nn.sch
Writing to Programs/Bytecode/mini_nn-0.bc
Hash: 6f6b0bf1ca7d18c18baa9378f00e4b960336744f42098204e46cc79a912e3798
Program requires at most:
           3 integer inputs from player 1
          21 integer inputs from player 0
         567 integer triples
        1862 integer bits
          22 integer opens
          56 virtual machine rounds
Compilation finished, running program...
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 0 mini_nn -pn 17239 -h localhost -N 2
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 1 mini_nn -pn 17239 -h localhost -N 2
Using statistical security parameter 40
Neural network output = 0.907394
The following benchmarks are including preprocessing (offline phase).
Time = 1.4714 seconds 
Data sent = 67.9008 MB in ~253 rounds (party 0 only; use '-v' for more details)
Global data sent = 135.801 MB (all parties)
```
## 4. Remarks

- We use approximations for nonlinear functions (common in MPC).  
- Matrix operations are more efficient than manual loops.  
- This is the foundation for private ML tasks like **secure classification** or **logistic regression**.  
