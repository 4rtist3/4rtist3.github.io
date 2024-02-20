---
title: Writeup GoogleCTF 2022
tags: CTF cryptography
cover: /assets/images/google/google.png
---

First time solving challenges from Google CTF, I feel pretty good but I have to learn more and more. 

| **Challenge** | **Solve** | **Note** |
|-----------|:-------:|:------:|
| ELECTRIC MAYHEM CLS | 70 solves | None |
| CYCLING | 50 solves | None |
| MAYBE SOMEDAY | 35 solves | Write later |

# ELECTRIC MAYHEM CLS
## Challenge
```
The server presents power traces of a secret firmware crypto operation. The goal is to recover the secret key.
Note, the flag is 'CTF{XXX}' where XXX is your recovered key.
```

At first, they give a lot of source code, which are written in C. Have a look at `main.c` we could see they use AES ECB to encrypt those plaintext.
```cpp
// Copyright 2022 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
#include <stdio.h>
#include <stdlib.h>

#include "aes.h"
#include "elmoasmfunctionsdef.h"

static uint32_t N = 1;
static uint8_t key[16] = {};
static uint8_t in[16] = {};

int main() {
  // Read #traces.
  LoadN(&N);

  // Read key.
  for (uint32_t j = 0; j < sizeof(key); j++) {
    readbyte(&key[j]);
  }

  struct AES_ctx ctx;
  AES_init_ctx(&ctx, key);

  for (uint32_t i = 0; i < N; i++) {
    // Read plaintext.
    for (uint32_t j = 0; j < sizeof(in); j++) {
      randbyte(&in[j]);
    }

    // Analyze AES encryption.
    starttrigger();
    AES_ECB_encrypt(&ctx, in);
    endtrigger();

    // Write ciphertext.
    for (uint32_t j = 0; j < sizeof(in); j++) {
      printbyte(&in[j]);
    }
  }

  endprogram();
  return 0;
}
```
## Solution
Look again with the web in the description, I see lots of plaintexts and each one have a graph to performs the trace.

![Trace each plaintext](/assets/images/google/gg1.png)

 
After a while, seeing a [**writeup**](https://ctftime.org/writeup/12148) with has a part similar to this challenge in Square CTF 2018. This is [**CPA manual attack**](https://wiki.newae.com/V4:Tutorial_B6_Breaking_AES_(Manual_CPA_Attack)?fbclid=IwAR2p7z0nehy7PsSTn2DT9HRSkWn9pT4Cdu3iCVJZ4pklROx3eBwKIz5Y1ic) in AES.

With some helping, finally I can get the data from the web. And the work is just change the plaintext and trace to what i got from the web.
```py
import requests
import numpy as np

res = requests.get("https://electric-mayhem-cls-web.2022.ctfcompetition.com/data")
#print(res.json()[0])
plaintexts = []
ciphertexts = []
sample = []
for i in range(50):
    print(i)
    temp = res.json()[i]
    pt, ct = temp["PT"], temp["CT"]
    plaintexts.append(np.array([int(pt[x:x+2], 16) for x in range(0,len(pt),2)]))
    ciphertexts.append(np.array([int(ct[x:x+2], 16) for x in range(0,len(ct),2)]))


for i in range(50):
    print(i)
    res = requests.get("https://electric-mayhem-cls-web.2022.ctfcompetition.com/data/{}".format(i))
    sample.append(np.array(res.json()))



np.save('plaintexts.npy',np.array(plaintexts)) 
np.save('ciphertexts.npy',np.array(ciphertexts))
np.save('sample.npy',np.array(sample))
```

Here is the code to get all data from web, I saved to 3 files, just in case.

```py

import base64
import binascii
import json
import re
import sys

import numpy as np

from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes


HW = [bin(n).count("1") for n in range(0,256)]

sbox=(
0x63,0x7c,0x77,0x7b,0xf2,0x6b,0x6f,0xc5,0x30,0x01,0x67,0x2b,0xfe,0xd7,0xab,0x76,
0xca,0x82,0xc9,0x7d,0xfa,0x59,0x47,0xf0,0xad,0xd4,0xa2,0xaf,0x9c,0xa4,0x72,0xc0,
0xb7,0xfd,0x93,0x26,0x36,0x3f,0xf7,0xcc,0x34,0xa5,0xe5,0xf1,0x71,0xd8,0x31,0x15,
0x04,0xc7,0x23,0xc3,0x18,0x96,0x05,0x9a,0x07,0x12,0x80,0xe2,0xeb,0x27,0xb2,0x75,
0x09,0x83,0x2c,0x1a,0x1b,0x6e,0x5a,0xa0,0x52,0x3b,0xd6,0xb3,0x29,0xe3,0x2f,0x84,
0x53,0xd1,0x00,0xed,0x20,0xfc,0xb1,0x5b,0x6a,0xcb,0xbe,0x39,0x4a,0x4c,0x58,0xcf,
0xd0,0xef,0xaa,0xfb,0x43,0x4d,0x33,0x85,0x45,0xf9,0x02,0x7f,0x50,0x3c,0x9f,0xa8,
0x51,0xa3,0x40,0x8f,0x92,0x9d,0x38,0xf5,0xbc,0xb6,0xda,0x21,0x10,0xff,0xf3,0xd2,
0xcd,0x0c,0x13,0xec,0x5f,0x97,0x44,0x17,0xc4,0xa7,0x7e,0x3d,0x64,0x5d,0x19,0x73,
0x60,0x81,0x4f,0xdc,0x22,0x2a,0x90,0x88,0x46,0xee,0xb8,0x14,0xde,0x5e,0x0b,0xdb,
0xe0,0x32,0x3a,0x0a,0x49,0x06,0x24,0x5c,0xc2,0xd3,0xac,0x62,0x91,0x95,0xe4,0x79,
0xe7,0xc8,0x37,0x6d,0x8d,0xd5,0x4e,0xa9,0x6c,0x56,0xf4,0xea,0x65,0x7a,0xae,0x08,
0xba,0x78,0x25,0x2e,0x1c,0xa6,0xb4,0xc6,0xe8,0xdd,0x74,0x1f,0x4b,0xbd,0x8b,0x8a,
0x70,0x3e,0xb5,0x66,0x48,0x03,0xf6,0x0e,0x61,0x35,0x57,0xb9,0x86,0xc1,0x1d,0x9e,
0xe1,0xf8,0x98,0x11,0x69,0xd9,0x8e,0x94,0x9b,0x1e,0x87,0xe9,0xce,0x55,0x28,0xdf,
0x8c,0xa1,0x89,0x0d,0xbf,0xe6,0x42,0x68,0x41,0x99,0x2d,0x0f,0xb0,0x54,0xbb,0x16)


def intermediate(pt, keyguess):
    return sbox[pt ^ keyguess]


def do_cpa(pt, traces):
    numtraces = np.shape(traces)[0]-1
    numpoint = np.shape(traces)[1]

    #Use less than the maximum traces by setting numtraces to something
    #numtraces = 10

    #Set 16 to something lower (like 1) to only go through a single subkey
    bestguess = [0]*16
    pge = [256]*16
    for bnum in range(0, 16):
        cpaoutput = [0]*256
        maxcpa = [0]*256
        for kguess in range(0, 256):

            #Initialize arrays &amp; variables to zero
            sumnum = np.zeros(numpoint)
            sumden1 = np.zeros(numpoint)
            sumden2 = np.zeros(numpoint)

            hyp = np.zeros(numtraces)
            for tnum in range(0, numtraces):
                hyp[tnum] = HW[intermediate(pt[tnum][bnum], kguess)]


            #Mean of hypothesis
            meanh = np.mean(hyp, dtype=np.float64)

            #Mean of all points in trace
            meant = np.mean(traces, axis=0, dtype=np.float64)

            #For each trace, do the following
            for tnum in range(0, numtraces):
                hdiff = (hyp[tnum] - meanh)
                tdiff = traces[tnum,:] - meant

                sumnum = sumnum + (hdiff*tdiff)
                sumden1 = sumden1 + hdiff*hdiff
                sumden2 = sumden2 + tdiff*tdiff

            cpaoutput[kguess] = sumnum / np.sqrt( sumden1 * sumden2 )
            maxcpa[kguess] = max(abs(cpaoutput[kguess]))

        bestguess[bnum] = np.argmax(maxcpa)

    return bestguess


def b64d(data):
    return base64.urlsafe_b64decode(str(data) + '=' * ((4 - len(data) % 4) % 4))


def main():
    pt = np.load('plaintexts.npy')
    traces = np.load('sample.npy')
    print((traces[0]))
    key = do_cpa(pt, traces)
    KEY = ""
    for i in key:
        i = i%256
        KEY+=hex(i)[2:]

    print(KEY)


if __name__ == '__main__':
    main()
```

And the key is : **`W0ckAwocKaWoCka1`**

**Flag: CTF{W0ckAwocKaWoCka1}** 
{:.success}

# CYCLING
## Challenge
```py
#!/usr/bin/python3

# Copyright 2022 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

"""
It is well known that any RSA encryption can be undone by just encrypting the
ciphertext over and over again. If the RSA modulus has been chosen badly then
the number of encryptions necessary to undo an encryption is small.
If n = 0x112b00148621 then only 209 encryptions are necessary as the following
example demonstrates:

>>> e = 65537
>>> n = 0x112b00148621
>>> pt = 0xdeadbeef
>>> # Encryption
>>> ct = pow(pt, e, n)
>>> # Decryption via cycling:
>>> pt = ct
>>> for _ in range(209):
>>>   pt = pow(pt, e, n)
>>> # Assert decryption worked:
>>> assert ct == pow(pt, e, n)

However, if the modulus is well chosen then a cycle attack can take much longer.
This property can be used for a timed release of a message. We have confirmed
that it takes a whopping 2^1025-3 encryptions to decrypt the flag. Pack out
your quantum computer and perform 2^1025-3 encryptions to solve this
challenge. Good luck doing this in 48h.
"""

e = 65537
n = 0x99efa9177387907eb3f74dc09a4d7a93abf6ceb7ee102c689ecd0998975cede29f3ca951feb5adfb9282879cc666e22dcafc07d7f89d762b9ad5532042c79060cdb022703d790421a7f6a76a50cceb635ad1b5d78510adf8c6ff9645a1b179e965358e10fe3dd5f82744773360270b6fa62d972d196a810e152f1285e0b8b26f5d54991d0539a13e655d752bd71963f822affc7a03e946cea2c4ef65bf94706f20b79d672e64e8faac45172c4130bfeca9bef71ed8c0c9e2aa0a1d6d47239960f90ef25b337255bac9c452cb019a44115b0437726a9adef10a028f1e1263c97c14a1d7cd58a8994832e764ffbfcc05ec8ed3269bb0569278eea0550548b552b1
ct = 0x339be515121dab503106cd190897382149e032a76a1ca0eec74f2c8c74560b00dffc0ad65ee4df4f47b2c9810d93e8579517692268c821c6724946438a9744a2a95510d529f0e0195a2660abd057d3f6a59df3a1c9a116f76d53900e2a715dfe5525228e832c02fd07b8dac0d488cca269e0dbb74047cf7a5e64a06a443f7d580ee28c5d41d5ede3604825eba31985e96575df2bcc2fefd0c77f2033c04008be9746a0935338434c16d5a68d1338eabdcf0170ac19a27ec832bf0a353934570abd48b1fe31bc9a4bb99428d1fbab726b284aec27522efb9527ddce1106ba6a480c65f9332c5b2a3c727a2cca6d6951b09c7c28ed0474fdc6a945076524877680
# Decryption via cycling:
pt = ct
for _ in range(2**1025 - 3):
  pt = pow(pt, e, n)
# Assert decryption worked:
assert ct == pow(pt, e, n)

# Print flag:
print(pt.to_bytes((pt.bit_length() + 7)//8, 'big').decode())
```
## Solution
At first, in the source code, they use [**cycle attack on RSA**](https://crypto.stackexchange.com/questions/1572/cycle-attack-on-rsa) which can be summarized as follows:

$$Compute \ {m^e} \ mod(n), \  m^{e^2} \ mod(n), \dots \ util  \ finding \ some \ k: m^{e^k} \ mod(n) = m \ mod(n)$$

Here, they use $c^{e^{2^{1025}-3}} \ mod(n)$ so that $h = 2^{1025} -3$ such that $m^{h+1} = m \ mod(n)$. If we use the loop or the equation, it will take such a very long time to calculate.

Look again, we have $m^{h+1} = m \ mod(n)$ corresponding $h + 1 = 1 \ mod(\lambda(n))$ ([**Euler's totient function**](https://en.wikipedia.org/wiki/Euler%27s_totient_function)), here $\lambda(n)$ is $Lcm(p-1, q-1)$. We have $Gcd(e, \lambda(n)) = 1$, so $h + 1$ is a multiple of $\lambda(\lambda(n))$.

Let have:

 $$\lambda(n) = \prod_{x = 1}^{k} p^{e_{x}}_{x}$$ 
 
 then:
 
$$\lambda(\lambda(n)) = \prod_{x = 1}^{k} \lambda(p^{e_{x}}_{x}) = \prod_{x = 1}^{k} p^{e_{x - 1}}_{x}(p_{x} - 1)$$
  
That means if we call $Z$ is a set divisors of $\lambda(\lambda(n))$, if $ isPrime(k + 1) : k \in Z , k$ probably is a prime factor of $\lambda(n)$.

```py
from factordb.factordb import FactorDB
k = 2**1025 - 3
f = FactorDB(k + 1)
f.connect()
stuff = f.get_factor_list()
# stuff = [2, 3, 5, 17, 257, 641, 65537,  274177,  2424833, 6700417, 67280421310721, 1238926361552897, 59649589127497217, 5704689200685129054721, 7455602825647884208337395736200454918783366342657, 93461639715357977769163558199606896584051237541638188580280321, 741640062627530801524787141901937474059940781097519023905821316144415759504705008092818711693940737]
```

Here is a prime factor of $k + 1$ with 17 primes as below we can generate 604 divisors $i$ which can be prime when take $(i + 1)$.

Let call 604 primes calculated are $P$, then calculate: 

$$\prod_{x = 1}^{k} a_{x} \ | \  a \in P$$

We will have $k* \lambda(n) \ (k \in N)$. Then we can find $d$ by $inverse(e,k*\lambda(n))$.

```py
from Crypto.Util.number import *
from factordb.factordb import FactorDB
import itertools

k = 2**1025 - 3
f = FactorDB(k + 1)
f.connect()
stuff = f.get_factor_list()
test = []
for L in range(1, len(stuff)+1):
    for subset in itertools.combinations(stuff, L):
        temp = 1
        for i in subset:
            temp*=i
        if isPrime(temp+1) and (temp+1) not in test:
            test.append(temp+1)


e = 65537
n = 0x99efa9177387907eb3f74dc09a4d7a93abf6ceb7ee102c689ecd0998975cede29f3ca951feb5adfb9282879cc666e22dcafc07d7f89d762b9ad5532042c79060cdb022703d790421a7f6a76a50cceb635ad1b5d78510adf8c6ff9645a1b179e965358e10fe3dd5f82744773360270b6fa62d972d196a810e152f1285e0b8b26f5d54991d0539a13e655d752bd71963f822affc7a03e946cea2c4ef65bf94706f20b79d672e64e8faac45172c4130bfeca9bef71ed8c0c9e2aa0a1d6d47239960f90ef25b337255bac9c452cb019a44115b0437726a9adef10a028f1e1263c97c14a1d7cd58a8994832e764ffbfcc05ec8ed3269bb0569278eea0550548b552b1
ct = 0x339be515121dab503106cd190897382149e032a76a1ca0eec74f2c8c74560b00dffc0ad65ee4df4f47b2c9810d93e8579517692268c821c6724946438a9744a2a95510d529f0e0195a2660abd057d3f6a59df3a1c9a116f76d53900e2a715dfe5525228e832c02fd07b8dac0d488cca269e0dbb74047cf7a5e64a06a443f7d580ee28c5d41d5ede3604825eba31985e96575df2bcc2fefd0c77f2033c04008be9746a0935338434c16d5a68d1338eabdcf0170ac19a27ec832bf0a353934570abd48b1fe31bc9a4bb99428d1fbab726b284aec27522efb9527ddce1106ba6a480c65f9332c5b2a3c727a2cca6d6951b09c7c28ed0474fdc6a945076524877680
phi = 1
for i in test:
    phi *= i 

d = inverse(e,phi)
print(long_to_bytes(pow(ct,d,n)))
```

**Flag: CTF{Recycling_Is_Great}** 
{:.success}
<!--more-->
