# False-Bottom Encryption: Deniable Encryption from Secret Sharing

A generic Python implementation of the False-Bottom Encryption scheme, supporting **any number of messages** with full encrypt, decrypt, update, and delete operations.

Based on the paper:

> **S. Ahmad, S. Rass, and P. Schartner**, "False-Bottom Encryption: Deniable Encryption From Secret Sharing," *IEEE Access*, vol. 11, pp. 62549–62564, 2023.
> DOI: [10.1109/ACCESS.2023.3288285](https://doi.org/10.1109/ACCESS.2023.3288285)

## What Is False-Bottom Encryption?

False-Bottom Encryption hides multiple plaintexts inside a single ciphertext using secret sharing. Different decryption keys reveal different messages from the same ciphertext. If you're coerced into handing over a key, you give a decoy key that decrypts to a fake message, while the real secret stays hidden.

The name comes from a stage magician's box: open the false bottom and it looks empty, but the real contents are underneath.

**Key properties:**

- **Plausible deniability** — multiple meaningful messages inside one ciphertext, each with its own key
- **Information-theoretic security** — no computational intractability assumptions, making the scheme post-quantum secure
- **Editable** — add, update, or delete messages within the ciphertext over time
- **Lightweight** — only field additions and multiplications, linear in the number of plaintexts

## How It Works

All arithmetic happens in $\mathbb{Z}_p$ for a prime $p$. The scheme works in three parts:

**Init:** Generate a random key base $\mathbf{r} = (r_1, \ldots, r_k) \in \mathbb{F}^*$ and an empty ciphertext $\mathbf{c} = (\alpha_1, \ldots, \alpha_k) \in \mathbb{F}^k$.

**Encrypt:** To add message $m_i$, represent it as a weighted sum of existing ciphertext elements and key base values:

$$m_i = \alpha_{i,1} \cdot r_{i,1} + \alpha_{i,2} \cdot r_{i,2} + \cdots + \alpha_{i,n_i} \cdot r_{i,n_i}$$

Pick $n_i - 1$ existing elements from $\mathbf{c}$, solve for a new $\alpha$, and append it. The secret key records which indices were used.

**Decrypt:** Given a key $sk_i$ (a list of index pairs), compute $m_i = \sum_{(j,\rho) \in sk_i} \mathbf{c}[j] \cdot \mathbf{r}[\rho] \mod p$.

## Installation

No dependencies beyond Python 3.8+. Just grab the file:

```bash
git clone https://github.com/<your-username>/false-bottom-encryption.git
cd false-bottom-encryption
```

## Usage

```python
from false_bottom_encryption import FalseBottomEncryption

# Initialize with prime p and key base size k
fbe = FalseBottomEncryption(p=9973, k=5)

# Encrypt any number of messages
sk_real   = fbe.encrypt(42)     # your real secret
sk_decoy1 = fbe.encrypt(1000)   # decoy message
sk_decoy2 = fbe.encrypt(7777)   # another decoy

# Decrypt with respective keys
print(fbe.decrypt(sk_real))     # 42
print(fbe.decrypt(sk_decoy1))   # 1000
print(fbe.decrypt(sk_decoy2))   # 7777

# Under coercion: hand over sk_decoy1 or sk_decoy2
# The adversary gets a valid-looking plaintext, not your real secret

# Update a message in-place
fbe.update(0, new_message=99)
print(fbe.decrypt(sk_real))     # 99

# Delete a message (overwrites with random data)
fbe.delete(1)
```

## Running

```bash
python false_bottom_encryption.py
```

You'll be asked to choose a mode:

```
False-Bottom Encryption
  1. Interactive mode (user prompts)
  2. Automated test (no input needed)
```

### Interactive Mode

Walks you through the full scheme step by step with prompts:

1. **System initialization** — choose the prime $p$ and key base size $k$
2. **Encryption** — specify how many messages to encrypt and enter each value
3. **Decryption** — automatically verifies all messages decrypt correctly
4. **Update** — specify how many messages to update, pick indices and new values
5. **Delete** — specify how many messages to delete, pick indices
6. **Final state** — shows the ciphertext, all keys, and verification results

Example session:

```
Enter the prime p (e.g. 9973): 9973
Enter the key base size k (e.g. 5): 5
How many messages do you want to encrypt? 5
  m_0 = 100
  m_1 = 200
  m_2 = 300
  m_3 = 42
  m_4 = 7777
  ...
  All decryptions correct!
How many messages do you want to update? (0 to skip): 1
  Which message index to update? [0-4]: 2
  New value for m_2 [0-9972]: 999
  Updated m_2: 300 -> 999
  All other messages unaffected!
How many messages do you want to delete? (0 to skip): 1
  Which message index to delete? 4
  Deleted m_4 (was 7777).
  Decryption with old key now returns: 8620 (garbage)
```

### Automated Test

Runs encrypt/decrypt on 10 messages, tests update and delete integrity, and scales up to 1000 messages:

```
  [Test 1] Encrypt/decrypt 10 messages: 10/10 correct
  [Test 2] Update m_2 -> 999: OK, others unaffected
  [Test 3] Delete m_4: OK (decrypts to garbage)
  [Test 4] Scale test (1000 messages, p=104729): 1000/1000 correct
```

## API Reference

### `FalseBottomEncryption(p, k, seed=None)`

Initialize the scheme.

- `p` — prime defining the field $\mathbb{Z}_p$ (default: 9973)
- `k` — size of the key base and initial ciphertext (default: 5)
- `seed` — optional RNG seed for reproducibility

### `encrypt(message) → SecretKey`

Add a message to the ciphertext. Returns the secret key needed for decryption. The ciphertext grows by exactly 1 element per call.

### `decrypt(sk) → int`

Recover a message given its secret key.

### `update(message_index, new_message)`

Replace the `message_index`-th message with `new_message` in-place. Other messages remain correctly decryptable.

### `delete(message_index)`

Overwrite a message with random data. Indistinguishable from an update to an observer.

### `get_state_summary() → str`

Return a human-readable dump of the current ciphertext, key base, and all stored keys.

## Parameters and Security

The prime $p$ determines the block size ($t \approx \lceil \log_2 p \rceil$ bits per message block). The key base size $k$ controls:

- How many terms can appear in each message's linear representation (between 2 and $k$)
- The initial ciphertext size before any messages are added
- The upper bound on brute-force complexity, which grows as $\Theta(k^{\dim(\mathbf{c})})$

For messages longer than $\lfloor \log_2 p \rfloor$ bits, split them into blocks and encrypt each block separately (analogous to ECB or other block cipher modes, as noted in the paper's Remark 1).

**Post-quantum security:** The scheme relies on information-theoretic arguments (entropy, underdetermined linear systems), not computational hardness assumptions. No factoring, no discrete logs, no lattice problems. A quantum computer does not help an attacker here.

## Limitations

- **No access pattern protection.** An adversary observing the ciphertext grow over time can count how many messages were added. The paper discusses ORAM-based extensions to address this.
- **Key management.** Updating or deleting a message requires knowledge of all secret keys (to ensure no collateral damage). This is inherent to the scheme's structure.
- **Single block per call.** Messages larger than $\lfloor \log_2 p \rfloor$ bits need to be split into blocks and encrypted individually.

## Citation

If you use this code in your research, please cite the original paper:

```bibtex
@article{ahmad2023falsebottom,
  author  = {Ahmad, Shahzad and Rass, Stefan and Schartner, Peter},
  title   = {False-Bottom Encryption: Deniable Encryption From Secret Sharing},
  journal = {IEEE Access},
  volume  = {11},
  pages   = {62549--62564},
  year    = {2023},
  doi     = {10.1109/ACCESS.2023.3288285}
}
```

## License

This code is provided for academic and research purposes. See the original paper's [Creative Commons BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/) license for terms governing the underlying work.
