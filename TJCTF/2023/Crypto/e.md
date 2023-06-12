## crypto/e

>Author: stosp
>
>64 solves / 60 points
>
>smol e

*Note: Below you'll find a guided solution. If interested just in the solve script, click [here](#solve-script-cryptoe)*

When we first open up our challenge, we are greeted with two files: `gen.sage` and `output.txt`. It's the former that interests us a lot more, so let's take a look:
```python
from Crypto.Util.number import bytes_to_long

p = random_prime(2 ^ 650)
q = random_prime(2 ^ 650)
N = p*q
e = 5
flag = open("flag.txt", "rb").read().strip()
m = bytes_to_long(b'the challenges flag is ' + flag)
c = m ^ e % N
print("N: ", N)
print("C: ", c)
print("e: ", e)
```
Obviously, we're dealing with RSA here. The primes seem safely generated and the encryption's standard - we can find our public key and the ciphertext in `output.txt`. So our eyes turn to the central section:
```python
e = 5
flag = open("flag.txt", "rb").read().strip()
m = bytes_to_long(b'the challenges flag is ' + flag)
```

Immediately we notice that $e = 5$. This should tell us that whatever attack we plan on mounting likely relies on a small exponent.

Next, we notice that our message (i.e. plaintext) is a combination of a known string and the (unknown) flag. More precisely, it means we have a partially known plaintext. 


#### Enter Coppersmith
Let's take a short detour.

In the mid-to-late 90s, a very clever man named Don Coppersmith, building on the ideas of other clever people, came up with a method for solving univariate modular equations. That is, equations with one variable modulo some $N$. He'd present a very efficient method for finding small roots (i.e. solutions) of such polynomials. In time, this would be generalized for multivariate polynomials as well. The details of the *how* are beyond the scope of this write-up (but can be found [here](https://cr.yp.to/bib/2001/coppersmith.pdf)).

Though that begs the question: what's *small*? For a polynomial of degree $k$, we call small roots of the polynomial those less than $N^{1/k}$. 

Now, what does this have to do with us? Well, let's look at our message:
```python
b'the challenges flag is ' + flag
```
Obviously, we know the higher bits of the plaintext ($kBits$) but don't know the flag ($x$). If we assume that the flag is $h$ bits large, we can rewrite our message as:

$$kBits * 2^{h} + x = m$$

Which tells us that the ciphertext is:

$$c \equiv m^e \equiv (kBits * 2^{h} + x)^e \ \ \ \ (mod\ N)$$

And since we know that $e=5$ we can write it as:

$$(kBits * 2^{h} + x)^5 - c \equiv 0  \ \ \ \ (mod\ N)$$

And what we get, after expanding it, is a:
- univariate polynomial (it has only one unknown, $x$),
- that is modulo $N$,
- with a small degree $k = e = 5$,
- and a relatively small root.

So we can apply Coppersmith! Luckily for us, [SageMath](https://www.sagemath.org/) already has Coppersmith's method implemented for us in the form of the `small_roots` function.

### Solve script (crypto/e)
To make our lives easier, we recall that the flag format for the CTF is `tjctf{` so we append that to our known string. Additionally, as standard calls to `small_roots()` don't result in any outputs, we increase the epsilon value (which defaults to 1/8). We can roughly understand it as increasing the bounds of the algorithm. The lower the epsilon, the larger the bounds.
```python
from sage.all import *

N = 8530080367614029604 ...
C = 2987003325076547237 ... 
e = 5
Z = Zmod(N)
P.<x> = PolynomialRing(Z)

msg = b'the challenges flag is tjctf{'
known = Z(int(msg.hex(), 16)) # Bytes to decimal

for unknown_bytes in range(2,33): # x < N^(1/5) ~= 32 bytes
    m = known * 2^(8*unknown_bytes)
    f = (m + x)^e - C
    
    roots = f.small_roots(epsilon=1/12)
    if roots != []:
        print(roots)
        break

```
When converting the number to bytes, we get `coppersword2}`, resulting in the full message being `the challenges flag is tjctf{coppersword2}`.