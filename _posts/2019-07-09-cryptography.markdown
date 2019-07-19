---
layout: post
title:  "Cryptography - Brief outline of some topics"
description: A few-lines description of a variety of cryptography topics
img:
date: 2019-07-09  +1225
---

I have had the opportunity to come across a very wide variety of cryptography topics which I've understood in very low depth. Each of these topics is extremely unique and could possibly solve a large variety of problems but unfortunately did not solve not the one I wanted, which leaves me with a list that I would love to remember one day when I do have a problem that they might solve.

For some topics, I have not done much research but I've put them on the list anyway since it goes with the theme.

Here goes...

## Popular Topics
### Homomorphic Encryption
Homomorphic encryption is a scheme that allows performing functions over encrypted data and finally get only the result without actually knowing the input values to the function.

Somewhat homomorphic encryption has been a very well solved problem. These schemes allow certain operations like infinite additions, some multiplications etc. Many of these schemes are also pretty practical like the BGN scheme etc.

Fully homomorphic encryption allows any function to be performed in such a manner. However, the biggest problem is that most FHE schemes are completely impractical (or atleast somewhat so, comparison operation takes 1s per comparison).

### Commitment Schemes
Commitment schemes are cryptographic schemes that are hiding and binding. They can later on be revealed and verified. The hiding/binding can be either computationally or perfectly.

Computationally binding/hiding schemes can be broken in exponential (non-polynomial time). It can be reduced to solving the discrete logarithm problem or equivalent.

Perfectly hiding/binding schemes cannot be broken even in non-polynomial time, i.e. even if the commiter has infinte computation power, it cannot open the value to a different amount (binding) or find out the hidden value (hiding).

*No commitment scheme can be both perfectly hiding and perfectly binding*.

It can be understood in this manner: if a scheme is perfectly hiding, the commitment must open to multiple values, else an attacker with infinite power can brute force to try all values in the input space and find out what the value opens to. Following this logic, if the commitment opens to multiple values, then the scheme is not perfectly binding to an untrusted committer.

The most popular commitment scheme is the Pedersen commitment scheme. This scheme can be in the Paillier system like g^x.h^r or in the ECC system liek x.G + r.H

### Oblivious Transfer
Sender sends N messages to the receiver. Receiver can read exactly 1 message without the sender knowing which one was read.

The scheme can also be extended to a T-of-N scheme

### Secret Sharing
N parties collaborate to compute a function on their secret inputs.

The N parties could be honest, honest-but-curious, malicious/adverserial, byzantine(colluding) etc.

#### Threshold secret sharing

#### Shamir's Secret Sharing Protocol

### Fair Exchange
A method by which A and B can exchange information (like digital signature, password etc.) in a fair manner, i.e. either both get the other's information or neither gets.

- Optimistic Fair Exchange: One party (originator) takes the risk of sending its item first, optimistically hoping that the other party will respons by sending its item. If other party does not send, involve the trusted third party to resolve the dispute.

### Millionaire's Problem
2 party protocol for the secure evaluation of function(x, y) = [x > y], where the inputs are in an encrypted form or are not known to the other parties.

### Secure Multiparty Computation
Generalized version of the millionaire's problem, where any function can be computed as an N-party protocol. Typically requires interaction and the N parties to be known beforehand.

The N parties should be able to compute the correct result with privacy even if some of them are adverserial/malicious and give wrong inputs.

How to Play any Mental Game or A Completeness Theorem for Protocols with Honest Majority.

In 19th ACM Symposium on Theory of Computing, 1987. Oded Goldreich, Silvio Micali, and Avi Wigderson.

### Threshold encryption and decryption schemes
Threshold scheme is a multi-party public key cryptosystem that requires T-of-N parties to encrypt/decrypt.

## Not-so-popular Topics
### Differential Privacy

### Order Preserving Encryption
An encryption scheme that allows a third party to perform comparisons on encrypted values. 

+ Involves a symmetric key encryption, where all the values are encrypted by the same party with the same symmetric key.
+ Useful for database related applications, where the database encrypts the values for computation on the cloud.

### Verifiable Encryption
Alice encrypts a value v which satisfies some cryptographic property M. Prove that the encryption is of a value satisfying the property.

The value being encrypted can be secret key, digital signature on an agreed message etc.

### Universally Composable Commitments

### Time Lapse Cryptography
Sender sends a message encrypted with a public key at time T which no one can decrypt until time T+del at which point everyone can form the private key and decrypt it.

Sounds somewhat similar to Proof-of-Work etc.

- [Time Lapse Cryptography](https://www.eecs.harvard.edu/~cat/tlc.pdf)

### Proxy Re-encryption
Convert ciphertext in one key to ciphertext in another key without performing a decryption followed by an encryption, revealing the decryption key to the final receiver through a proxy.

Untrusted party(proxy) can convert messages between keys without access to the message or the decryption keys.

In the simplest form, suppose encryption key of first party is k_1 and decryption key is k_1^{-1} and similarly for party2. Then, party 1 gives the proxy the value k_1^{-1} . k2 so that the message is directly decryptable and readable by party 2. It is also cheaper than one decryption and one encryption.

#### Conditional Proxy Re-encryption

### Spooky Encryption

### Verifiable Computation

### Deniable Encryption

### Lossy Encryption
A lossy encryption scheme is a public key encryption scheme which produces real keys with which encryptions are committing and "lossy" keys with whcih encryptions are non-committing.

## Miscellaneous
### Pallier Cryptosystem
Paper: [Public-Key Cryptosystems Based on Composite Degree Residuosity Classes](https://www.cs.tau.ac.il/~fiat/crypt07/papers/Pai99pai.pdf)

It is a set of cryptography schemes based on the Decisional Composite Residuosity Assumption.

The assumption is that it is intractable to distinguish/find n-th residues modulo n^2.
