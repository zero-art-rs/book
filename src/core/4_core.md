# Cortado Curve
```admonish note
Our implementation of argument systems defined in the framework heavily uses our fork of Dalek’s [bulletproofs](https://github.com/zkcrypto/bulletproofs) implementation:

[https://github.com/zk-messenger/bulletproofs](https://github.com/zk-messenger/bulletproofs)
```

Having to prove statements about points over elliptic curves, efficiency is crucial for our applications, so we found elliptic curve **Cortado $E/\mathbb{F}_q:y^2=x^3+ax+b, \#E(\mathbb{F}_q)=r$** that is defined over scalar field $\mathbb{F}_q$ of [Ristretto](https://ristretto.group/) elliptic curve group (built over the same parameters as [curve25519](https://cr.yp.to/ecdh.html)), which is used as a core group for _bulletproofs_ implementation. So **Cortado** forms so-called semi-cycle of elliptic curves with _Ristretto._ We use **Cortado** curve as a main ART group $\mathbb{G}$ in our implementation.
```admonish note
[https://github.com/zk-messenger/project-m/blob/feature-creds/crates/zk/src/curve/cortado/cortado.rs](https://github.com/zk-messenger/project-m/blob/feature-creds/crates/zk/src/curve/cortado/cortado.rs)
```

## Parameters

```
q = 0x1000000000000000000000000000000014def9dea2f79cd65812631a5cf5d3ed
```

```
r = 0x100000000000000000000000000000007694c45f377088ccabbd72861f7cba8b
```

```
a = 3
```

```
b = 0x44046d0b327ef5023f5204c6a00ffd21ef176b641795bb22c68ac9d3bdc9d18
```

Base points (could be seen as a Pedersen basis):

```
G = (0xb4e65b863c4d2f65041c91f0eda3147bc7b288c0994c9833eba34f0979cf416, 0xb6ed55f428dcdb8fde1419dc53084b70c10c9da632511b9407c6b5d7defc6e4)
```

```
H = (0x2043492057a5d4ee72a746140818b194949275ce144fc994203703d5c14d1a8, 0x6612dddfb454452dfda7fcc761369d451e3cd045b81b9679be5f745c607d8a6)
```

## Security considerations
Curve is generated randomly by the sage script:
```python
# Define field F_p
p = 0x1000000000000000000000000000000014def9dea2f79cd65812631a5cf5d3ed
Fp = FiniteField(p)
print("Поле F_p, де p =", p)

# Function to find a curve with a simple order
def find_prime_order_curve(max_attempts=10000000):
    b = Fp.random_element()
    for i in range(max_attempts):
        # Generate random coefficients a and b in F_p
        a = 3
        b += 1
        # Checking the oddness condition: 4a^3 + 27b^2 ≠ 0
        if 4*a^3 + 27*b^2 == 0:
            continue  # The curve is singular, skip
        # Creating an elliptic curve in Weierstrass-form
        E = EllipticCurve(Fp, [a, b])
        # Calculate the order of E(F_p)
        n = E.cardinality()
        if i % 1000 == 0:
            print("Attempt ", i, " b =", b)
        if E.is_supersingular():
            print("Supersingular curve")
            continue
        # Checking if the order is simple
        if is_prime(n):
            print("\\nA curve with a simple order is found after", i+1, "attempts:")
            print("a =", a)
            print("b =", b)
            print("Curve order =", n)
            return E
    print("\\nNo curve with prime order found after", max_attempts, "attempts.")
    return None

# Curve search
E = find_prime_order_curve()

if E is not None:
    print("\\nResulting curve:")
    print(E)
```

The curve has prime order of length 253bits and MOV attack is not applicable so that the curve gives nearly 126bits of security under _Discrete Logarithm_ assumption.