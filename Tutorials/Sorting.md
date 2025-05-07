# Sorting in Secure Multi-Party Computation (MPC)

Sorting is a fundamental operation in computer science. Whether you're processing data for analytics, searching for values, or ensuring order for algorithms like binary search, sorting is everywhere.
But in the context of **secure multi-party computation (MPC)**, even basic tasks like sorting become non-trivial.

This tutorial explains how to perform sorting securely in MPC: why traditional algorithms do not work, and how to use two classic algorithms in the MPC setting using the [MP-SPDZ](https://github.com/data61/MP-SPDZ) framework. We assume that the reader is familiarized with the very basics of MP-SPDZ, for example by reading [documentation](https://mp-spdz.readthedocs.io/en/latest/readme.html).

---

## The challenge

In MPC, multiple parties compute jointly on private data without revealing their individual inputs. Each value is **secret-shared**, meaning no single party can see the actual plaintext value.

A traditional sorting algorithm may perform operations like this:

```python
if x[i] > x[j]:
    swap(x[i], x[j])
```

This is a **problem in MPC** because the condition `x[i] > x[j]` is evaluated **in the clear**, and the `if` statement leaks information about the data ordering, and consequently the inputs.
Hence, sorting in MPC cannot perform operations such as conditional branches based on secret values, revealing result of comparisons or having secret bounds on loop iterations.
Instead, one must use data-independent control flow and secure conditional swaps that are **oblivious** to the data being operated upon.

---

## Secure sorting

Sorting in MPC is performed using **oblivious sorting algorithms**, where the sequence of operations (comparisons and swaps) is fixed in advance, independent of input data. Two well-known examples are:

* **Bubble Sort** (educational, but inefficient)
* **Odd-Even Mergesort** (efficient and practical for MPC)

These algorithms use **secure comparisons** and **conditional swaps** to avoid leaking information.

In frameworks like MP-SPDZ, this is typically done using `cond.if_else(a, b)`:

```python
cond = x > y          # Secure comparison result
min_val = cond.if_else(y, x)
max_val = cond.if_else(x, y)
```

---

## Bubble Sort in MPC

Bubble Sort is one of the simplest sorting algorithms. It repeatedly steps through the list, compares adjacent elements, and swaps them if they are in the wrong order. While intuitive, it has poor performance due to its `O(n^2)` time complexity, for datasets containing `n` elements.

In the MPC setting, we can implement Bubble Sort using **nested loops and secure compare-and-swap logic**.

```python
def bubble_sort(secret_list):
    n = len(secret_list)
    for i in range(n):
        for j in range(n - i - 1):
            a = secret_list[j]
            b = secret_list[j + 1]
            cond = a > b
            secret_list[j] = cond.if_else(b, a)
            secret_list[j + 1] = cond.if_else(a, b)
```

---

## Odd-Even Mergesort in MPC

Odd-Even Mergesort is a parallel, recursive [sorting network](https://en.wikipedia.org/wiki/Batcher_odd%E2%80%93even_mergesort). It's based on the idea of **recursively sorting halves of the array** and then merging them using a fixed compare-and-swap pattern called the **odd-even merge**. This makes it ideal for secure computation because its pattern is predictable and independent of data.

Odd-Even Mergesort has a much better asymptotic complexity of `O(n log^2 n)`, and it scales well for large inputs in MPC. In fact, it is one of the sorting algorithms implemented within MP-SPDZ.

```python
def compare_and_swap(x, i, j):
    a = x[i]
    b = x[j]
    cond = a > b
    x[i] = cond.if_else(b, a)
    x[j] = cond.if_else(a, b)

def odd_even_merge(x, lo, n, r):
    step = r * 2
    if step < n:
        odd_even_merge(x, lo, n, step)
        odd_even_merge(x, lo + r, n, step)
        for i in range(lo + r, lo + n - r, step):
            compare_and_swap(x, i, i + r)
    else:
        compare_and_swap(x, lo, lo + r)

def odd_even_mergesort_rec(x, lo, n):
    if n > 1:
        m = n // 2
        odd_even_mergesort_rec(x, lo, m)
        odd_even_mergesort_rec(x, lo + m, m)
        odd_even_merge(x, lo, n, 1)

def odd_even_mergesort(x):
    odd_even_mergesort_rec(x, 0, len(x))
```

## Experimental results

To demonstrate the trade-offs between these algorithms, let's try a little demo. After implementing the algorithms in MP-SPDZ, we will compile and run two sorting programs with inputs of 128 elements:

```python
n = 128  # must be a power of 2 to keep odd-merge-sort simple
x = [sint.get_input_from(0) for _ in range(n)]  # secret input from party 0
sorting_algorithm(x) # odd_even_mergesort or bubble_sort
for i in range(n):
    print_ln("%s", x[i].reveal())
```

Notice how secret-shared values need to be opened to all parties using `reveal` before they can be printed to check that the output is really sorted. We will use a small piece of code in Python to populate the array for the first party:

```python
from random import randint
 
with open("Player-Data/Input-P0-0", "w") as file:
    file.writelines(f"{randint(1, 100)}\n" for _ in range(128))
```

Now let's compile and run both programs on `localhost` using the [MASCOT}(https://eprint.iacr.org/2016/505) backend, and look at the performance results. Starting with `bubble-sort.mpc`:
```bash
./compile.py bubble-sort.mpc
Scripts/mascot.sh -v bubble-sort
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 0 bubble-sort -pn 16954 -h localhost -N 2
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 1 bubble-sort -pn 16954 -h localhost -N 2
Using statistical security parameter 40
The following benchmarks are including preprocessing (offline phase).
Time = 242.044 seconds 
Data sent = 21954.3 MB in ~29247 rounds (party 0 only; use '-v' for more details)
Global data sent = 43888.1 MB (all parties)
```

And now with `odd-merge-sort.mpc`

```bash
./compile.py odd-even-mergesort.mpc
Scripts/mascot.sh -v odd-even-mergesort
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 0 odd-even-mergesort -pn 17594 -h localhost -N 2
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../mascot-party.x 1 odd-even-mergesort -pn 17594 -h localhost -N 2
Using statistical security parameter 40
...
The following benchmarks are including preprocessing (offline phase).
Time = 69.8483 seconds 
Data sent = 6557.69 MB in ~8023 rounds (party 0 only; use '-v' for more details)
Global data sent = 13094.9 MB (all parties)
```

One can immediately see the much lower number of multiplications and openings in the second approach, leading to huge savings in time and communication.
