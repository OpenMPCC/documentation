
# Optimized Secure Edit Distance in MPC

This tutorial explains an **optimized secure computation** of the **edit distance** between two parties using [MP-SPDZ](https://github.com/data61/MP-SPDZ).
It shows how conversion between Boolean and arithmetic shares can improve performance, and illustrates how **edaBits** in MP-SPDZ can be used for efficient comparisons.
It is based on the paper [Privacy-Preserving Edit Distance Computation Using Secret-Sharing Two-Party Computation](https://eprint.iacr.org/2023/1201), published in [Latincrypt 2023](https://www.espe.edu.ec/latincrypt/).
Previous papers proposed approaches based on homomorphic encryption and garbled circuits, this time MPC based on secret sharing is the tool of choice.

---

# Algorithm in plaintext

The [edit distance](https://en.wikipedia.org/wiki/Edit_distance) (also known as Levenshtein distance) is a metric for how related two strings of data are.
It gives the number of operations (insertions, deletions and substitutions) needed to go from string _A_ to string _B_. It is commonly used in genome sequencing and DNA analysis.

The [Wagner–Fischer algorithm](https://en.wikipedia.org/wiki/Wagner%E2%80%93Fischer_algorithm) formalizes the problem as a recursion, from which a **dynamic programming** solution can be efficiently implemented.
The core idea of the algorithm is building a matrix `D` of `size (len(A)+1)` × `(len(B)+1)`, where `D[i][j]` is the minimum edit distance between:
* the first `i` characters of string _A_
* and the first `j` characters of string _B_.

```python
import numpy as np


def edit_distance(A, B):
    D = np.zeros(shape=(len(A) + 1, len(B) + 1))

    # Fill base cases
    for i in range(len(A) + 1):
        D[i][0] = i # (cost of deleting all characters from A)
    for j in range(len(B) + 1):
        D[0][j] = j # (cost of deleting all characters from B)

    for i in range(1, len(A) + 1):
        for j in range(1, len(B) + 1):
            # Compare two characters of the string
            if A[i - 1] == B[j - 1]:
                t = 0
            else:
                t = 1

            D[i][j] = min(
                    D[i - 1][j] + 1,       # Deletion
                    D[i][j - 1] + 1,       # Insertion
                    D[i - 1][j - 1] + t    # Substitution
                    )

    # Not return the final edit distance
    return D[len(A)][len(B)]                


if __name__ == "__main__":
    file_chain_A = open("Inputs/arithmetic/Input-P0-0")
    file_chain_B = open("Inputs/arithmetic/Input-P1-0")

    A = file_chain_A.read().replace("\n", "")
    B = file_chain_B.read().replace("\n", "")

    file_chain_A.close()
    file_chain_B.close()

    print("Distance =", edit_distance(A, B))
```

## Example

Given `A = "kitten"` and `B = "sitting"`, the DP matrix looks like this:

```
      ''  s  i  t  t  i  n  g
   -------------------------
'' | 0  1  2  3  4  5  6  7
k  | 1  1  2  3  4  5  6  7
i  | 2  2  1  2  3  4  5  6
t  | 3  3  2  1  2  3  4  5
t  | 4  4  3  2  1  2  3  4
e  | 5  5  4  3  2  2  3  4
n  | 6  6  5  4  3  3  2  3
```

The final edit distance is `D[6][7] = 3`.


# Edit distace in MPC

Genomic data is frequently large and highly sensitive, so finding a way to efficiently compute the edit distance metric in a privacy preserving way is quite relevant.
In the case of genomic data, the characters in _A_ and _B_ are nucleotides (`ATCG`) which can be encoded using only 2 bits each.
However, note that the algorithm has two different types of operations:
* Comparisons between characters from _A_ and _B_ and entries of the matrix `D`
* Arithmetic between entries of `D` and 1 or the result of a comparison
This means the MPC approach will involve **mixed arithmetic**, which can be optimized by using [extended doubly-authenticated bits](https://eprint.iacr.org/2020/338) or _edaBits_.

## What Is an edaBit?

An _edaBit_ (efficiently generated bit-decomposed value) is a special type of preprocessed value in MP-SPDZ that provides:
* A secret-shared number `x`, and
* Its bit-decomposition `[x_0, x_1, ..., x_{k-1}] where x = x₀ + 2x₁ + ... + 2^{k-1}x_{k-1}`
Both the value and its bits are already secret-shared, generated during the preprocessing phase.
This means we can switch from the two representations efficiently.

## Algorithm in MPC

We split the algorithm above in two stages, as to minimize the number of conversions. First we compare all characters of `A` and `B` in bitwise representation and store the results of the comparisons in a matrix.
Next we convert the results of these comparisons to secret-shared integers, and execute the dynamic programming approach in MPC.
The resulting code computes the edit distance  between two secret-shared strings A and B, meaning each party holds inputs and does not reveal them to others. The function securely computes how many insertions, deletions, or substitutions are needed to turn one string into the other.
The full implementation can be found [here](https://github.com/hdvanegasm/sec-edit-distance/blob/master/Source/Secure/optim_edit_distance.mpc). Let's go through it step by step.

**1. Comparisons**

Enable _edaBits_ in the backend protocol, a secret-shared bit type and functions to compare two individual nucleotides.
The comparison function amounts to computing the XOR between the pairs of bits in each of the nucleotides to compare them, and the XOR between the results of the two comparisons.
We negate the result such that it outputs `1` if the nucleotides are equal.

```python
program.use_edabit(True)
sb = sbits.get_type(1)

def xor_nucleotids(N1, N2):
    xor = Array(2, sb)
    xor[0] = N1[0] ^ N2[0]
    xor[1] = N1[1] ^ N2[1]
    return xor


def or_nucleotid(N):
    return N[0].bit_or(N[1])


def equal_nucleotids(N1, N2):
    return or_nucleotid(xor_nucleotids(N1, N2)).bit_not()
```

**2. Comparison matrix**

Compute and store all the results of comparisons in a matrix.

```python
def compute_comp_matrix(A, B):
    M_comp = Matrix(len(A), len(B), sb)
    @for_range_opt(len(A))
    def _(i):
        @for_range_opt(len(B))
        def _(j):
            comp = equal_nucleotids(A[i], B[j]).bit_not()
            M_comp[i][j] = comp

    return M_comp
```

**3. Main loop**

With the matrix `T` of comparisons already computed, we convert the results of comparisons to secret-shared integers and the rest of the algorithm is relatively straightforward, being quite similar to the plaintext version above.
Notice that we use `@for_range_opt` to execute loops depending on bounds that are provided in the command line, and thus are not known at compile-time.

```python
def edit_distance(A, B):
    T_bits = compute_comp_matrix(A, B)
    T = Matrix(len(T_bits), len(T_bits[0]), sint)

    @for_range_opt(len(T_bits))
    def _(i):
        @for_range_opt(len(T_bits[0]))
        def _(j):
            T[i][j] = sint(T_bits[i][j])
    
    D = Matrix(len(A) + 1, len(B) + 1, sint)
    D.assign_all(0)

    @for_range_opt(len(A) + 1)
    def _(i):
        D[i][0] = i

    @for_range_opt(len(B) + 1)
    def _(j):
        D[0][j] = j

    @for_range(1, len(A) + 1)
    def _(i):
        @for_range(1, len(B) + 1)
        def _(j):
            min_terms = [
                D[i - 1][j] + 1, 
                D[i][j - 1] + 1, 
                D[i - 1][j - 1] + T[i - 1][j - 1]
            ]
            D[i][j] = util.min(min_terms)
    
    return D[len(A)][len(B)]
```

**4. Reads inputs and execute the program**

The final missing piece is reading strings from the inputs given by the two parties, calling the MPC function and printing the final result.

```python
def read_strings(A_length, B_length):
    A = Matrix(A_length, 2, sb)
    B = Matrix(B_length, 2, sb)

    for i in range(A_length):
        A[i][0] = sb.get_input_from(0)
        A[i][1] = sb.get_input_from(0)
    for j in range(B_length):
        B[j][0] = sb.get_input_from(1)
        B[j][1] = sb.get_input_from(1)

    return A, B

A_length = int(program.args[1])
B_length = int(program.args[2])
A, B = read_strings(A_length, B_length)
d = edit_distance(A, B)
print_ln("Edit distance = %s", d.reveal())
```

## Experimental results

Let's now try this idea in practice. After implementing the algorithms in MP-SPDZ, we will compile and run two sorting programs with input strings of 100 nucleotids each.
The backend of choice will be `spdz2k` as in the original paper, which can be accelerated with edaBits:

```bash
./compile.py edit-distance.mpc 100 100
Scripts/spdz2k.sh -v optim_edit_distance-100-100
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../spdz2k-party.x 0 -v optim_edit_distance-100-100 -pn 11237 -h localhost -N 2
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../spdz2k-party.x 1 -v optim_edit_distance-100-100 -pn 11237 -h localhost -N 2
Using SPDZ2k security parameter 64
Using statistical security parameter 40
Trying to run 64-bit computation
Edit distance = 52
The following benchmarks are including preprocessing (offline phase).
Time = 92.3301 seconds 
Data sent = 6775.89 MB in ~738945 rounds (party 0 only)
Global data sent = 13551.8 MB (all parties)
```

One runtime optimization we can make is reduce the precision of the arithmetic secret-sharings.
Because our inputs are bounded in length, we can actually use less than 64 bits to represent shares, for example 16.
We can make this modification by rebuilding the MP-SPDZ backends with lower precision and running the benchmarks again:

```bash
make RING_SIZE=16 spdz2k-party.x
./compile.py -R 16 edit-distance.mpc 100 100
Scripts/spdz2k.sh -v optim_edit_distance-100-100
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../spdz2k-party.x 0 optim_edit_distance-100-100 -pn 13321 -h localhost -N 2
Running /home/dfaranha/projects/MP-SPDZ/Scripts/../spdz2k-party.x 1 optim_edit_distance-100-100 -pn 13321 -h localhost -N 2
Using SPDZ2k security parameter 64
Using statistical security parameter 40
Trying to run 16-bit computation
Edit distance = 52
The following benchmarks are including preprocessing (offline phase).
Time = 35.8789 seconds 
Data sent = 2041.55 MB in ~559161 rounds (party 0 only; use '-v' for more details)
Global data sent = 4083.09 MB (all parties)
