## crypto/keysmith

>Author: stosp
>
>30 solves / 128 points
>
>I lost my key... can you find it?
>
>`nc tjc.tf 31103`

*Note: Below you'll find a guided solution. If interested in just the solve script, click [here](#solve-script-cryptokeysmith)*

Downloading the challenge, we stumble upon `server.py` - let's see what we've got:
```python
#!/usr/local/bin/python3.10 -u
from Crypto.Util.number import getPrime
flag = open("flag.txt", "r").read()

po = getPrime(512)
qo = getPrime(512)
no = po * qo
eo = 65537

msg = 762408622718930247757588326597223097891551978575999925580833
s = pow(msg,eo,no)

print(msg,"\n",s)

try:
    p = int(input("P:"))
    q = int(input("Q:"))
    e = int(input("E:"))
except:
    print("Sorry! That's incorrect!")
    exit(0)

n = p * q
d = pow(e, -1, (p-1)*(q-1))
enc = pow(msg, e, n)
dec = pow(s, d, n)
if enc == s and dec == msg:
    print(flag)
else:
    print("Not my keys :(")
```

Interesting. So we're given $m$ and $s$ (and technically $e_o$), and nothing else. Let's put this into some equations:

$$n_o = p_o * q_o$$
$$s \equiv m^{e_o} \quad (mod\ n_o)$$

So our $s$ is essentially an encrypted message. And for our chosen input of $e$, $p$ and $q$ we have:

$$n = p * q$$

$$d \equiv e^{-1} \quad (mod\ n)$$

$$enc \equiv m^e \quad (mod\ n),\ \text{where $enc$ should be the same as $s$}$$

$$dec \equiv s^d \quad (mod\ n),\ \text{where $dec$ should be the same as $m$}$$

Right, that's a few expressions. Let's put some words behind them.

Effectively, we're looking to input parameters $n$ and $e$ such that $m^e$ equals to a specific number $s$. If our numbers $p$ and $q$ are both prime, then the second test regarding $dec$ will always pass.

Obviously, one way to get the flag would be to send the private parameters back. However, seeing as the target $n_o$ is rather large, factoring it isn't really an option.
So what now? Well, let's think back to what we just said: we're looking to find an $e$ such that $m^e$ equals to $s$. This problem, when stated over a numbers modulo $n$, is called the **discrete logarithm problem** (or DLP for short).

The DLP is the 'hard' problem behind cryptosystems such as Diffie-Hellman. It's proven to be rather difficult in the general case, however there are some special cases when it's far easier. Notably, for a prime $p$, if $p-1$ is smooth (meaning it has many prime factors), it's possible to solve the DLP in polynomial time. But is us being forced to send two primes, resulting in a composite DLP, going to be an issue? Of course not! Thanks to the Chinese Remainder Theorem, we can solve the DLP modulo each prime factor using the efficient algorithm.

The algorithm in question, called [Pohlig–Hellman](https://en.wikipedia.org/wiki/Pohlig%E2%80%93Hellman_algorithm), is luckily implemented in SageMath through the `discrete_logarithm` function. All that's left now is to code-up the rest.

#### Generating $p-1$ smooth numbers
While for solving our special case of the DLP we have an efficient algorithm, how would we go about generating these 'smooth' numbers? The method we'll employ is surprisingly simple: multiply a lot of small numbers, check if the incremented version of the number is prime and, if so, return that number. Here's one way to achieve just that in SageMath:
```python
from sage.all import *

def smooth_prime(r=80): 
    not_found = True 
    while not_found: 
        k = 2 
        for _ in range(r): 
            k *= random_prime(1000, 10000) * random_prime(10000, 1000000) 
        k += 1 
        if is_prime(k) and len(factor(k-1)) > 20: 
            not_found = False 
            return k 
```
Here our function's parameter `r` corresponds to a rough estimate of how many prime multiplications we'd like. While not ideal, it gets the job done - a better method would be to go about multiplying random numbers until a certain bit-length is reached. But we don't have to worry about a key-size here, as long as our public key $n$ is larger than $s$,  we're set.

### Solve script (crypto/keysmith)

To simplify things, we only bother with ensuring $p-1$ is smooth. For $q$ we can send any small prime factor. It should be noted that we might need multiple attempts with this script; sometimes, we'll get such an $s$ that finding a modular inverse is unlikely (or impossible).
```python
from sage.all import *

def smooth_prime(r=80): 
    not_found = True 
    while not_found: 
        k = 2 
        for _ in range(r): 
            k *= random_prime(1000, 10000) * random_prime(10000, 1000000) 
        k += 1 
        if is_prime(k) and len(factor(k-1)) > 20: 
            not_found = False 
            return k 
                         
# === SETUP
from pwn import *
server = "tjc.tf"
port = 31103
io = remote(server, port)

# === INIT
msg = int(io.recvline())
s = int(io.recvline())

eo = 65537

# === PAYLOAD
successful = False
while not successful:
    p = int(smooth_prime(55))
    q = 3
    n = p*q
    print("Attempting...")
    Z = Zmod(n)
    try:
        e = discrete_log(Z(s), Z(msg))
        e = int(e)
        e = e % ((p-1)*(q-1))
        d = pow(e, -1, (p-1)*(q-1))
    except ValueError:
        continue
    except ZeroDivisionError:
        continue
    
    successful = True

# === CHECKS
enc = pow(msg, e, n)
dec = pow(s, d, n)
assert enc == s
assert dec == msg

# === ATTACK
p_ = str(p).encode()
print(p_)
io.sendline(p_)

q_ = str(q).encode()
print(q_)
io.sendline(q_)

e_ = str(e).encode()
print(e_)
io.sendline(e_)

io.interactive() # tjctf{lock-smith_289378972359}
```