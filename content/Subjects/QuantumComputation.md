---
title: Quantum Computation
aliases:
  - QutuamComputation
id: 41A504A9-DFF6-4CA8-9531-E56BCFDFC1C7
---

# Quantum Computation

These notes introduce qubit states, gates, entanglement, quantum teleportation, and the Deutsch–Jozsa algorithm.

## 1. Notation and single-qubit states

### Kets, bras, and inner products

A **ket** is a column vector. Its corresponding **bra** is its conjugate transpose, denoted by $\dagger$:

$$
|a\rangle = \begin{pmatrix}a_1 \\ a_2\end{pmatrix},
\qquad
\langle b| = |b\rangle^\dagger
= \begin{pmatrix}b_1^* & b_2^*\end{pmatrix}.
$$

The **inner product** is a complex number:

$$
\langle b|a\rangle = b_1^*a_1 + b_2^*a_2.
$$

The **outer product** $|a\rangle\langle b|$ is a matrix, representing an operator.

### Computational basis and normalization

The computational basis consists of

$$
|0\rangle = \begin{pmatrix}1 \\ 0\end{pmatrix},
\qquad
|1\rangle = \begin{pmatrix}0 \\ 1\end{pmatrix}.
$$

A pure single-qubit state has the form

$$
|\psi\rangle = \alpha|0\rangle + \beta|1\rangle,
\qquad
\alpha,\beta\in\mathbb{C},
\qquad
|\alpha|^2 + |\beta|^2 = 1.
$$

The last condition is normalization: $\langle\psi|\psi\rangle=1$.

### Other useful basis states

The $X$ basis is

$$
|+\rangle = \frac{|0\rangle+|1\rangle}{\sqrt{2}},
\qquad
|-\rangle = \frac{|0\rangle-|1\rangle}{\sqrt{2}}.
$$

The $Y$ basis is

$$
|+i\rangle = \frac{|0\rangle+i|1\rangle}{\sqrt{2}},
\qquad
|-i\rangle = \frac{|0\rangle-i|1\rangle}{\sqrt{2}}.
$$

The factor $1/\sqrt{2}$ normalizes each of these equal-weight superpositions.

## 2. Single-qubit gates and measurements

### Pauli gates

The Pauli matrices are unitary gates and Hermitian observables:

$$
X=\sigma_x=\begin{pmatrix}0&1\\1&0\end{pmatrix},
\qquad
Y=\sigma_y=\begin{pmatrix}0&-i\\i&0\end{pmatrix},
\qquad
Z=\sigma_z=\begin{pmatrix}1&0\\0&-1\end{pmatrix}.
$$

- **Bit flip:** $X|0\rangle=|1\rangle$ and $X|1\rangle=|0\rangle$.
- **Phase flip:** $Z|0\rangle=|0\rangle$ and $Z|1\rangle=-|1\rangle$; consequently, $Z|+\rangle=|-\rangle$.
- **Bit and phase flip:** $Y=iXZ$, so $Y|0\rangle=i|1\rangle$ and $Y|1\rangle=-i|0\rangle$.

In particular, $Y|+i\rangle=|+i\rangle$ and $Y|-i\rangle=-|-i\rangle$: these are eigenstates of $Y$, so $Y$ does not exchange them.

### Hadamard and phase gates

The **Hadamard gate** changes between the computational and $X$ bases:

$$
H=\frac{1}{\sqrt{2}}\begin{pmatrix}1&1\\1&-1\end{pmatrix},
\qquad
H|0\rangle=|+\rangle,
\qquad
H|1\rangle=|-\rangle.
$$

Since $H^2=I$, it also maps $|+\rangle$ to $|0\rangle$ and $|-\rangle$ to $|1\rangle$.

The **phase gate** is

$$
S=\begin{pmatrix}1&0\\0&i\end{pmatrix},
\qquad
S|+\rangle=|+i\rangle,
\qquad
S|-\rangle=|-i\rangle.
$$

Here $S$ denotes a gate; the subscript $S$ later denotes a separate qubit in the teleportation protocol.

### Projective measurements

For a normalized state $|\psi\rangle$ and an orthonormal measurement basis $\{|x_k\rangle\}$, the **Born rule** gives

$$
p(k)=\left|\langle x_k|\psi\rangle\right|^2.
$$

The modulus is essential because amplitudes can be complex. After outcome $k$, the state is $|x_k\rangle$, up to an overall phase.

Measuring a Pauli observable means measuring in its eigenbasis:

| Observable | Eigenstate for outcome $+1$ | Eigenstate for outcome $-1$ |
| --- | --- | --- |
| $Z$ | $\lvert0\rangle$ | $\lvert1\rangle$ |
| $X$ | $\lvert+\rangle$ | $\lvert-\rangle$ |
| $Y$ | $\lvert+i\rangle$ | $\lvert-i\rangle$ |

A computational-basis measurement commonly reports the bit labels $0$ and $1$, corresponding to the $Z$ eigenvalues $+1$ and $-1$. Applying a Pauli gate and measuring that Pauli observable are different operations.

## 3. Multiple qubits and entanglement

### Tensor products

Composite systems are described using the tensor product $\otimes$:

$$
|a\rangle\otimes|b\rangle
=
\begin{pmatrix}a_1\\a_2\end{pmatrix}
\otimes
\begin{pmatrix}b_1\\b_2\end{pmatrix}
=
\begin{pmatrix}a_1b_1\\a_1b_2\\a_2b_1\\a_2b_2\end{pmatrix}.
$$

We abbreviate $|a\rangle\otimes|b\rangle$ as $|a\rangle|b\rangle$ or $|ab\rangle$. Throughout these notes, the two-qubit basis order is

$$
|00\rangle,\ |01\rangle,\ |10\rangle,\ |11\rangle.
$$

A bipartite **pure state** is a product state if it can be written as $|a\rangle\otimes|b\rangle$; otherwise, it is entangled. For mixed states, classical correlations can exist without entanglement, so correlation alone does not define entanglement.

The symbol $\oplus$ below denotes addition modulo two (XOR), not a tensor product.

### Controlled-NOT (CNOT)

With the first qubit as control and the second as target,

$$
\operatorname{CNOT}|x\rangle|y\rangle
=|x\rangle|y\oplus x\rangle,
\qquad x,y\in\{0,1\}.
$$

In the basis order above,

$$
\operatorname{CNOT}=
\begin{pmatrix}
1&0&0&0\\
0&1&0&0\\
0&0&0&1\\
0&0&1&0
\end{pmatrix}.
$$

```text
control: |x⟩ ──●── |x⟩
               │
target:  |y⟩ ──⊕── |y ⊕ x⟩
```

### Bell states

The four Bell states form an orthonormal basis of maximally entangled two-qubit states:

$$
\begin{aligned}
|\varphi^{00}\rangle &= \frac{|00\rangle+|11\rangle}{\sqrt{2}},\\
|\varphi^{01}\rangle &= \frac{|01\rangle+|10\rangle}{\sqrt{2}},\\
|\varphi^{10}\rangle &= \frac{|00\rangle-|11\rangle}{\sqrt{2}},\\
|\varphi^{11}\rangle &= \frac{|01\rangle-|10\rangle}{\sqrt{2}}.
\end{aligned}
$$

To prepare $|\varphi^{ij}\rangle$ from $|i\rangle_A|j\rangle_B$, apply $H$ to $A$, then CNOT with control $A$ and target $B$:

$$
|\varphi^{ij}\rangle
=\operatorname{CNOT}_{A\to B}(H\otimes I)|ij\rangle.
$$

```text
A: |i⟩ ──H──●──
            │
B: |j⟩ ─────⊕──
```

## 4. Quantum teleportation

### Setup

Alice holds a qubit $S$ in the unknown state

$$
|\psi\rangle_S=\alpha|0\rangle_S+\beta|1\rangle_S.
$$

Alice and Bob also share $|\varphi^{00}\rangle_{AB}$: Alice holds $A$, and Bob holds $B$. In qubit order $S,A,B$, the joint state is

$$
\begin{aligned}
|\Psi\rangle
&=|\psi\rangle_S\otimes|\varphi^{00}\rangle_{AB}\\
&=\frac{1}{\sqrt{2}}\left(
\alpha|000\rangle+\alpha|011\rangle
+\beta|100\rangle+\beta|111\rangle
\right).
\end{aligned}
$$

### Expansion in Alice's Bell basis

Regrouping the same state in the Bell basis of $S,A$ gives

$$
\begin{aligned}
|\Psi\rangle=\frac{1}{2}\Bigl(
&|\varphi^{00}\rangle_{SA}\otimes|\psi\rangle_B\\
+{}&|\varphi^{01}\rangle_{SA}\otimes X|\psi\rangle_B\\
+{}&|\varphi^{10}\rangle_{SA}\otimes Z|\psi\rangle_B\\
+{}&|\varphi^{11}\rangle_{SA}\otimes XZ|\psi\rangle_B
\Bigr).
\end{aligned}
$$

The prefactor is $1/2$ because the Bell states are already normalized. Each Bell outcome has probability $1/4$.

### Measurement and correction

1. Alice measures $S,A$ in the Bell basis, obtaining the label $(i,j)$.
2. Alice sends the two classical bits $i,j$ to Bob.
3. Bob applies $Z^iX^j$ to recover $|\psi\rangle$.

Operator products act from right to left: Bob applies $X^j$ first, then $Z^i$.

| Alice's outcome $(i,j)$ | Bob's state before correction | Bob's correction |
| --- | --- | --- |
| $(0,0)$ | $\lvert\psi\rangle$ | $I$ |
| $(0,1)$ | $X\lvert\psi\rangle$ | $X$ |
| $(1,0)$ | $Z\lvert\psi\rangle$ | $Z$ |
| $(1,1)$ | $XZ\lvert\psi\rangle$ | $ZX$ |

Alice implements the Bell measurement by applying $\operatorname{CNOT}_{S\to A}$, then $H$ on $S$, then measuring both qubits in the computational basis. The outcomes on $S$ and $A$ are $i$ and $j$, respectively.

```text
S: |ψ⟩ ──●──H──measure Z → i
         │
A: ──────⊕─────measure Z → j
B: ──────────────────────────X^j──Z^i── |ψ⟩
```

Here $A,B$ start in a shared Bell state; Bob's gates depend on the received classical bits. Teleportation consumes that entanglement and does not leave Alice with a copy of the unknown state. See [IBM Quantum Learning: Quantum teleportation](https://quantum.cloud.ibm.com/learning/en/courses/basics-of-quantum-information/entanglement-in-action/quantum-teleportation) for a circuit-based derivation.

## 5. Oracles and Hadamard transforms

### Bit oracle and phase oracle

For a Boolean function $f:\{0,1\}^n\to\{0,1\}$, a **bit oracle** acts as

$$
O_f|x\rangle|y\rangle=|x\rangle|y\oplus f(x)\rangle.
$$

A **phase oracle** acts on the input register as

$$
U_f|x\rangle=(-1)^{f(x)}|x\rangle.
$$

To obtain the phase-oracle action using the bit oracle, prepare the target qubit in $|-\rangle$:

$$
O_f\bigl(|x\rangle\otimes|-\rangle\bigr)
=(-1)^{f(x)}|x\rangle\otimes|-\rangle
=(U_f|x\rangle)\otimes|-\rangle.
$$

This is **phase kickback**. The target remains $|-\rangle$; the same phase-oracle action does not follow for an arbitrary target state $|y\rangle$.

### Hadamard on one qubit

For $x\in\{0,1\}$,

$$
H|x\rangle=\frac{1}{\sqrt{2}}
\sum_{k\in\{0,1\}}(-1)^{kx}|k\rangle.
$$

### Hadamard on an n-qubit register

For a bit string $x\in\{0,1\}^n$,

$$
H^{\otimes n}|x\rangle
=\frac{1}{\sqrt{2^n}}
\sum_{k\in\{0,1\}^n}(-1)^{k\cdot x}|k\rangle,
$$

where $k\cdot x=\sum_{r=1}^n k_rx_r\pmod 2$ is the binary inner product. In particular,

$$
H^{\otimes n}|0\rangle^{\otimes n}
=\frac{1}{\sqrt{2^n}}\sum_x|x\rangle.
$$

## 6. Deutsch–Jozsa algorithm

### Problem and promise

Given oracle access to $f:\{0,1\}^n\to\{0,1\}$, with $n\geq1$, determine whether $f$ is:

- **Constant:** the same output for every input.
- **Balanced:** output $0$ for exactly half the inputs and $1$ for the other half.

The algorithm assumes that one of these two cases holds.

### Circuit and procedure

Using the phase oracle from the previous section:

```text
n-qubit register: |0⟩^⊗n ──H^⊗n──U_f──H^⊗n──measure
```

1. Initialize the register to $|0\rangle^{\otimes n}$.
2. Apply $H$ to every qubit.
3. Apply $U_f$ once.
4. Apply $H$ to every qubit again.
5. Measure in the computational basis. The all-zero result means constant; any other result means balanced.

With a bit oracle, use one extra target qubit initialized to $|1\rangle$ and apply $H$ to prepare $|-\rangle$. One call to $O_f$ then implements the needed phase kickback. See [IBM Quantum Learning: The Deutsch–Jozsa Algorithm](https://quantum.cloud.ibm.com/learning/en/modules/computer-science/deutsch-jozsa).

### Derivation

Write $0^n$ for the all-zero bit string. The successive register states are

$$
|\psi_0\rangle=|0^n\rangle=|0\rangle^{\otimes n},
$$

$$
|\psi_1\rangle=H^{\otimes n}|\psi_0\rangle
=\frac{1}{\sqrt{2^n}}\sum_{x\in\{0,1\}^n}|x\rangle,
$$

$$
|\psi_2\rangle=U_f|\psi_1\rangle
=\frac{1}{\sqrt{2^n}}\sum_{x\in\{0,1\}^n}(-1)^{f(x)}|x\rangle.
$$

After the second Hadamard transform,

$$
\begin{aligned}
|\psi_3\rangle
&=\frac{1}{2^n}\sum_x\sum_k(-1)^{f(x)+k\cdot x}|k\rangle\\
&=\sum_{k\in\{0,1\}^n}C_k|k\rangle,
\end{aligned}
$$

where

$$
C_k=\frac{1}{2^n}\sum_{x\in\{0,1\}^n}(-1)^{f(x)+k\cdot x}.
$$

The probability of measuring the all-zero string is therefore

$$
P(0^n)=|\langle0^n|\psi_3\rangle|^2
=|C_{0^n}|^2
=\left|\frac{1}{2^n}\sum_x(-1)^{f(x)}\right|^2.
$$

- If $f$ is constant, all terms have the same sign, so $C_{0^n}=\pm1$ and $P(0^n)=1$.
- If $f$ is balanced, positive and negative terms cancel, so $C_{0^n}=0$ and $P(0^n)=0$.

Thus, under the promise and with ideal operations, one oracle query distinguishes the two cases with certainty. Without the promise, this measurement rule does not classify arbitrary functions as constant or balanced.

## 7. Grover's algorithm

*To be developed: the original notes contained only a heading for this topic.*
