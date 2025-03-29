---
title: "Learning Python exceptions through Lenstra's algorithm"
date: 2022-03-30
tags:
  - programming
  - computational number theory
# draft: true
---

> [!note]
> This is part two of my series on Lenstra's Algorithm. See part one [here](https://nathanielbd.github.io/posts/algorithm-built-to-fail/) for the intuition for the algorithm.

---

In part one, we saw that Lenstra's Algorithm can intuitively be understood as an algorithm literally built to fail. 
We repeatedly add a point to itself on an elliptic curve modulo the number to be factored and wait until a failure
to compute a modular multiplicative inverse occurs. Because when that happens, it means that the GCD of the number
to be factored and another number is non-zero. The latter number must be a factor of the former! We'll see that this
pattern of waiting for a failure to occur and handling it is exactly what exceptions help us do in the Python programming
language. Along the way, let's also dig deeper into the subroutines and implementation details of the algorithm.

# What are exceptions?

```python
try:
    y = 1/x
except ZeroDivisionError:
    print("Can't divide by zero!")
```

In the above code block, if `x` is ever `0`, we will 'throw' a `ZeroDivisionError` exception. When an exception occurs
within a `try` block, it must be 'caught' with an `except` block. When the exception is caught, the code within the 
`except` block is executed instead of the code within the `try` block.

What's even more interesting is the behavior in Python if there isn't an `except` block immediately after the `try` block.

```python
import math

def invert(x):
    return 1/x

def quadratic_formula(a, b, c):
    try:
        determ = b**2 - 4*a*c
        over_neg_two_a = invert(2*a)
        return (-b + math.sqrt(determ))*over_neg_two_a, (-b - math.sqrt(determ))*over_neg_two_a
    except ZeroDivisionError:
        print("That isn't even a quadratic function, silly!")

quadratic_formula(0, 3, 2)
```

If we run this code, we first call `quadratic_formula`. It will then call `invert` in its `try` block. Next, `invert` will end up throwing a `ZeroDivisionError` because
`0` is passed in as `x`. `invert` doesn't have an `except` block, so we look for what function called `invert`, which is `quadratic_formula`. 

`quadratic_formula` actually does have an `except` block! Our exception is finally caught.

# Let's program Lenstra's algorithm in Python

We seek to factor the integer $N$ using trying different points $P = (a, b)$ on the elliptic curve

$$
E: Y^2 = X^3 + AX + B \mod N
$$

where $B = b^2 - a^3 - Aa \mod N$. Suppose we will try $k$ different points for now.

```python
def lenstras(N, A, a, b, k=100):
    P = (a, b)
    for i in range(k):
        try:
            # compute the next point Q by adding P to itself
            # throw a NonUnityGCDEvent exception when a GCD is not 1
        except NonUnityGCDEvent:
            # catch the exception and return the nontrivial factor of N
        P = Q
```

As stated in part 1, it's actually faster to compute $Q = n!P$ for the $n$th iteration of the loop instead of repeated addition $Q = P + P + \dots P = nP$.

## The double-and-add algorithm

For $n! = m_0 \cdot m_1 \dots m_{n+1}$, the fastest way to compute each $m_i P$ is via the double-and-add algorithm. We'll repeat it to compute $n!$.

The basic idea is to use the binary expansion of $m_i$.

$$
m_i = r_0 + r_1 \cdot 2 + r_2 \cdot 4 + r_3 \cdot 8 + \dots r_j 2^j.
$$

Take $n=42$ as an example. Represented in binary, it's 101010.

$$
\begin{aligned}
    n &= 2^5 + 0 + 2^3 + 0 + 2^1 + 0 \newline
    &= 32 + 8 + 2\newline
    &= 42.
\end{aligned}
$$

How can we turn the evaluation of this expansion into an algorithm?

| ![](/DoubleAdd.mp4) |
|:--:|
| A walkthrough of the algorithm on the example. Notice how this uses 8 additions instead of the 41 additions required from the naive approach of computing $nP$. Can you justify why this algorithm has a time complexity of $O(\log_2(n))$ versus $O(n)$ for the naive approach? [Animation code](https://gist.github.com/nathanielbd/f45970af570c74dc9676a3fd4b9b728c) |

```python
def double_add(n, P):
    Q = P
    R = 0
    while n > 0:
        # could alternatively do 
        # n & 1
        # if you're into the bitwise and operator
        if bin(n)[-1] == '1': 
            # we'll replace this with elliptic curve point addition later
            R = R + Q 
        # again, this will be replaced with point addition
        Q = 2*Q
        # shift the bits right one
        n = n >> 1
    return R # R = nP
```

Alternatively, we can implement the algorithm without bitshifting and a binary representation altogether.

```python
import math

def double_add(n, P):
    Q = P
    R = 0
    while n > 0:
        # we're really just checking if the number is odd
        if n % 2 == 1:
            R = R + Q
        Q = 2*Q
        # the bitshift is equivalent to halving and flooring
        n = math.floor(n/2)
    return R
```

### Elliptic curve point addition

The algorithm for point addition is given in [part 1](https://nathanielbd.github.io/posts/algorithm-built-to-fail/#elliptic-curve-addition-algorithm). To summarize, we solve for the third intersection between the secant line and the elliptic curve. If it's a point doubling, we solve for the second intersection between the tangent line and the elliptic curve. Otherwise, we handle special cases with the identity element at infinity.

| ![](/PointAddition.mp4) |
|:--:|
| Recall point addition for the usual case of two distinct points. |

As the focus of this post is to illustrate how Python exceptions match our intuition of how the algorithm handles failures, let's simply translate the algorithm into Python without delving into the details outlined in the previous post.

```python
def pt_add(P, Q, A):
    if P == 0:
        return Q
    if Q == 0:
        return P
    x_p, y_p = P
    x_q, y_q = Q
    # special case where the points share an x-coordinate
    # the secant line is vertical, so the identity element is 
    # considered the intersection
    if x_p == x_q and y_p == -y_q:
        return 0
    # point doubling
    if P == Q:
        # division will require finding modular multiplicative
        # inverse with the extended Euclidean algorithm
        lam = (3*x_p**2 + A)/(2*y_1)
    # distinct points
    else:
        # again, we will replace division with a call to
        # the extended Euclidean algorithm
        lam = (y_q - y_p)/(x_q - x_p)
    x_r = lam**2 - x_1 - x_2
    y_r = lam*(x_p - x_r) - y_p
    R = (x_r, y_r)
    return R
```

### The extended Euclidean algorithm

[In part 1](https://nathanielbd.github.io/posts/algorithm-built-to-fail/#extended-euclidean-algorithm), we discuss a memory-inefficient implementation of the extended Euclidean algorithm. Let's revisit the example to see why it's inefficient.

$$
21u + 71v = \gcd(21, 71)
$$

| Divide, find remainder, repeat | Construct Bézout coefficients $u$ and $v$ |
|:--:|:--:|
| $\begin{aligned} 71 &= 3 \cdot 21 + 8 \newline 21 &= 2 \cdot 8 + 5 \newline 8 &= 1 \cdot 5 + 3 \newline 5 &= 1 \cdot 3 + 2 \newline 3 &= 1 \cdot 2 + 1 \newline 2 &= 2 \cdot 1 + 0 \newline \end{aligned}$ | $\begin{aligned} 8 &= 1 \cdot 71 + -3 \cdot 21 \newline 5 &= 0 \cdot 71 + 1 \cdot 21 - 2 (1 \cdot 71 + -3 \cdot 21) = -2 \cdot 71 + 7 \cdot 21 \newline 3 &= 1 \cdot 71 + -3 \cdot 21 - 1 (-2 \cdot 71 + 7 \cdot 21) = 3 \cdot 71 + -10 \cdot 21 \newline 2 &= -2 \cdot 71 + 7 \cdot 21 - 1 (3 \cdot 71 + -10 \cdot 21) = -5 \cdot 71 + 17 \cdot 21 \newline 1 &= 3 \cdot 71 + -10 \cdot 21 - 1 (-5 \cdot 71 + 17 \cdot 21) = 8 \cdot 71 + -27 \cdot 21 \end{aligned}$ |
| In part 1, we went over the procedure of dividing by the remainder repeatedly. This results in a decreasing sequence of remainders. Instead of storing the entire sequence, we can get away with only storing the two most recently computed remainders. | Part 1 also considers a second calculation of rewriting each remainder step in order to construct the coefficients. Not only can we again store only the most recently computed coefficients, but we can also compute them in parallel with the remainders instead of after. |


```python
def extended_euclidean(a, b):
    prev_r, r = a, b
    prev_u, u = 1, 0
    prev_v, v = 0, 1

    while r > 0:
        # integer/floor division
        quotient = prev_r // r
        # algebra required to solve for the variable
        # at each iteration, using parallel assignment
        prev_r, r = r, prev_r - quotient*r
        prev_u, u = u, prev_u - quotient*u
        prev_v, v = v, prev_v - quotient*v
    
    # since the loop will terminate on a zero 
    # remainder, return the coefficients before that
    return prev_u, prev_v
```

Revisiting the same example...

| Step | `quotient` | `prev_r` | `r` | `prev_u` | `u` | `prev_v` | `v` |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| 0 | undefined | 71 | 21 | 1 | 0 | 0 | 1 |
| 1 | 3 | 21 | 8 | 0 | 1 | 1 | -3 |
| 2 | 2 | 8 | 5 | 1 | -2 | -3 | 7 |
| 3 | 1 | 5 | 3 | -2 | 3 | 7 | -10 |
| 4 | 1 | 3 | 2 | 3 | -5 | -10 | 17 |
| 5 | 1 | 2 | 1 | -5 | 8 | 17 | -27 |
| 6 | 2 | 1 | 0 | 8 | -13 | -27 | 44 |

Notice that the `r` column is decreasing. It terminates at 0, consistent with our while loop logic. Comparing to the previous table with our naive approach, we can see the same coefficients being computed.

There's another optimization we can do that will allow us to save memory by not storing `prev_v` and `v`, since we can solve for the second coefficient after computing the first. We will leave that aside for now.

#### Computing a modular multiplicative inverse

Next, we'll do the necessary modifications to the extended Euclidean algorithm for it to compute modular multiplicative inverses. Recall from part 1 that we can only compute the modular multiplicative inverse of $a \mod b$ if $\gcd(a, b) = 1$.

$$
\begin{aligned}
au + bv &= \gcd(a, b) \newline
au + bv &= 1 \newline
au - 1 &= -bv \newline
au - 1 &\equiv 0 \mod b \newline
au &\equiv 1 \mod b.
\end{aligned}
$$

Now we can apply the final optimization, since we only need one coefficient anyways.

```python
def mod_mult_inverse(a, b):
    prev_r, r = b, a
    prev_inv, inv = 0, 1

    while r > 0:
        quotient = prev_r // new_r
        prev_inv, inv = inv, prev_inv - quotient*inv
        prev_r, r = r, prev_r - quotient*r

    if prev_r > 1:
        # the last non-zero remainder will be our GCD
        # if it is not one, we fail to compute the inverse
        # raise an exception to be handled upstream in
        # order to report the failure and capture our
        # non-trivial factor    
        raise NonUnityGCDEvent(prev_r) # <-- Exception starts here (1)
        # notice we pass prev_r (the non-unity GCD and non-trivial
        # factor) as extra information
    if inv < 0:
        # the coefficient might be negative
        # in this case, find its equivalent modulo b
        inv += b

    return inv

# create our own custom exception
class NonUnityGCDEvent(Exception):
    """
    Exception to be raised during computations in Lenstra's algorithm.
    Attributes:
        d -- integer GCD 1 < d <= N resulting from computation of a modular multiplicative inverse
    """
    def __init__(self, d, message="Found non-unity GCD d"):
        self.d = d
        self.message = message
        super().__init__(self.message)
```

# Conclusion

Finally, we can see the rest of the Python code with modifications and trace the event back up.

```python
def pt_add(P, Q, A, N):
    if P == 0:
        return Q
    if Q == 0:
        return P
    x_p, y_p = P
    x_q, y_q = Q
    if x_p == x_q and y_p == -y_q:
        return 0
    # point doubling
    if P == Q:
        # division now replaced with finding the modular
        # multiplicative inverse
        lam = (3*x_p**2 + A)*mod_mult_inverse(2*y_1, N) # <-- Exception rises here, and continues to rise as there is no `except` block (2)
    # distinct points
    else:
        # again division replaced with the modular
        # multiplicative inverse
        lam = (y_q - y_p)*mod_mult_inverse(x_q - x_p, N) # <-- Or rises here, depending on where it was called (2)
    x_r = lam**2 - x_1 - x_2
    y_r = lam*(x_p - x_r) - y_p
    R = (x_r, y_r)
    return R

def double_add(n, P, A, N):
    Q = P
    R = 0
    while n > 0:
        if n & 1:
            # replaced addition operator with elliptic
            # curve point addition function
            R = pt_add(R, Q, A, N) # <-- Exception rises here next, as it called pt_add (3)
        # replaced multiplication by two with elliptic
        # curve point addition function
        Q = pt_add(Q, Q, A, N) # <-- Or exception rises here next, if we were in the middle of point doubling when the non-unity GCD was found (3)
        n = n >> 1
    return R

def lenstras(N, A, a, b, k=100):
    P = (a, b)
    for i in range(k):
        try:
            Q = double_add(i, P, A, N) # <-- Exception rises to the top of the call stack, where there is an `except` block (4)
        except NonUnityGCDEvent e:
            # catch the exception and return the non-trivial factor of N
            return e.d
        P = Q
    # computation of k!P did not result in a non-trivial factor
    return None
```

You can see that the exceptions allow the computer implementation of the algorithm to follow the same flow as you would if you were performing Lenstra's algorithm by hand; you would go down elliptic curve/modular arithmetic rabbit hole until you found the factor by failure, remembered what led you down that particular hole, and immediately stop. An implementation without exceptions would require unnecessary returning of a flag indicating failure through all of the helper functions, then checking it at the top of the call stack and computing the GCD. It doesn't evoke the intuition of computing by failure or the thought process of a human performing the algorithm at all.

# Anticipated questions

What if you don't find any factors after $k$ tries? Or if $d = N$?
- You pick a different curve. There are [a few factors](https://en.wikipedia.org/wiki/Lenstra_elliptic-curve_factorization#Why_does_the_algorithm_work?) that affect how likely it is to encounter a division failure during calculations, but they are out of scope of this already quite long blog post. The bottom line is picking a curve and starting point gives us another chance at finding a subgroup with a number of elements that's "smooth" ---has small prime factors--- which will allow us to get a division failure quicker.

> [!info]
> I learned this algorithm from an excellant textbook, 'An Introduction to Mathematical Cryptography' by Hoffstein, Pipher, and Silverman. You can view Python implementations of selected algorithms from that textbook, including the one we just went over, [here](https://github.com/nathanielbd/HPS-crypto)