# Z3dAdventures

We're given a Python script implementing a stream cipher and an output file containing an 80-bit Initialization Vector (IV) and the resulting ciphertext. The goal is to recover the flag hidden at the end of a long diary entry.

The encryption script source.py uses a custom stream cipher inspired by Trivium. Key characteristics include:

- State Size: 177 bits.
- Key/IV Size: 80 bits each.
- Non-linearity: The feedback logic uses AND gates (&), making it a non-linear feedback shift register (NLFSR).
- Vulnerability: We are provided with a large amount of Known Plaintext (the diary entry). In a stream cipher, if you know the plaintext P and the ciphertext C, you can recover the key K using K = P ^ C.

Because the internal state is small (177 bits) and the unknown key is only 80 bits, we can model the cipher's logic as a system of equations and solve for the key using an SMT solver.

## Solution

I solved this using Python with the Z3 theorem prover (z3-solver) and the bitstring library.

1. Recovering the keystream

First, we XOR the known diary text with the provided ciphertext to extract the first few hundred bits of the keystream:

```
known_pt = b"Dear Diary...\nI have a super secret too, though: "
keystream = [int(ciphertext[i]) ^ int(known_pt[i]) for i in range(len(known_pt))]
```

2. Z3

We define the 80-bit key as symbolic variables in Z3. We then initialize the cipher's state exactly as the script does, but using these variables instead of concrete bits.

```
from z3 import *
key_bits = [Bool(f'k_{i}') for i in range(80)]
# State = Key (80) + Constant (13) + IV (80) + Padding (4)
state = key_bits + padding + iv_bits + [False]*4
```

3. Solving for the key

The solver needs to: 
	- Simulate the 708 "warmup" steps (where no output is produced) to mix the key and IV symbolically
	- For the next 200 bits, we tell Z3 that the symbolic output bit must equal the keystream bit we recovered in Step 1.

```
for i in range(200):
    state, out_bit = step_symbolic(state)
    solver.add(out_bit == (keystream[i] == 1))
```

4. Decryption

Once solver.check() returns sat, we extract the concrete values for the 80 key bits. With the key recovered, we re-run the original cipher logic to decrypt the entire ciphertext, revealing the flag:
`FLAG{n0w_U_kn0w_z3_t00_w4snt_1t_fun?}`

--------

I hope this helps, have a nice day :)
