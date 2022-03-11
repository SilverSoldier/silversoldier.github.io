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

There are a lot, by which I mean a LOT of MPC protocols out there, ranging from supporting honest majority to dishonest majority, semi-honest adversaries to malicious adverseries and son on.

[MP-SPDZ](https://github.com/data61/MP-SPDZ) is one of the best open-source code bases that I have found for any practical MPC implementations.

#### Links and References
1. How to Play any Mental Game or A Completeness Theorem for Protocols with Honest Majority.
2. 

### Threshold encryption and decryption schemes
Threshold scheme is a multi-party public key cryptosystem that requires T-of-N parties to encrypt/decrypt.

## Not-so-popular Topics
### Differential Privacy

### Order Preserving Encryption/Order Revealing Encryption
An encryption scheme that allows a third party to perform comparisons on encrypted values. 

+ Involves a symmetric key encryption, where all the values are encrypted by the same party with the same symmetric key.
+ Useful for database related applications, where the database encrypts the values for computation on the cloud.
+ Mostly only found references [on Stanford's crypto page]()

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

## India Cryptography Meet
Some topics and notes from the India Crypto Meet organized on 16.07.20. Contains some high level topics for future references and notes when I was able to follow and type at the same time.

The notes could be better organized, but the important point is that there are quite a few new and interesting techniques that were mentioned in the talk, which hopefully I have managed to note down.

### Reverse Firewall (Chaya Ganesh)
Preserving security even when honest parties' machines are subverted. We include corrupt implementations.

Reverse firewall which santitizes inputs and outputs of a machine.

It should maintain functionality of protocol. Output of protocol must remain the same.

Firewall is not a trusted party.

A corrupted implementation along with a reverse firewall is expected to maintain the same security as a non corrupted implementation. Protocol with firewall is secure for corrupt implementation of Alice.

#### Reverse Firewalls for Interactive Proofs
- Subversion resistance in Non-interactive zero knowledge
  - Security under parameter subversion, still assumes trusted implementation.

- Zero Knowlege Proofs:
	- Completeness: Honest verifier is convinced of a true statement
	- Soundness: Dishonest prover cannot prove honest verifier of incorrect statement.
	- Zero Knowledge: Cheating verifier cannot learn anything more than truth of statement
	- Special Soundness:
	- Special HVZK

Subverted Prover can leak bits of witness, which a Reverse Firewall cannot stop.

Functionality-maintaining tempering: protocol is still functional when prover replaced with subverted prover.

General firewall for all sigma protocols:
randomize prover's first message (in case it sends witness etc.)

- Malleable Sigma Protocol

- Security Preservation: ZK Preservation
- Exfiltration resistance: Protect implementation leaking secrets.

#### OR Transform (Cramer, Damgard ..CDS94)
Interesting technique ???

### Differential Privacy (Manoj Prabhakaran)
Dwork, McSherry, Nissim, Smith TCC 2006

The talk defines Differential Privacy as a success story in privacy techniques for statistical databases and talks about some limitations and how they can be addressed.

Mechanism to guard data, guarantee privacy while allowing statistical computations on it.

Only information revealed is information that would be revealed even if the individual does not contribute to database.
Information that might have been learned without the individual's contribution.

Stronger definition: Cannot even tell if a given individual has contributed to the database. (Membership itself is private, for instance medical trial related datasets).

Output of mechanism is an approximation of the desired function.

High level definition is to add some noise to the data. Amount of noise to add depends on the sensitivity of the function.

Typically, we use the Laplace Mechanism, where the noise is a laplacian curve.
Other noise distributions exist such as smooth sensitivity, exponential, randomized response (no central mechanism to add noise, each individual themselves add some noise to the response).

Adding noise does come with its limitations especially for high sensitivity functions, where adding noise destroys utility. One example of this is the max function.

To address this, we introduce a notion of accuracy while the privacy notion is unchanged.

Secondly, privacy cannot be guaranteed across databases. For instance, if we use one of 2 different databases, we might reveal which database was used.

To address this, we introduce a more robust definition of privacy.

#### Flexible Accuracy
PAC: Probably Approximately Correct

Probability that the approximation deviates too much from the actual value is bounded.

We define a close dataset D* to the original dataset D if it can be obtained by dropping a few elements (outliers).
This is a more reasonable definition of a neighboring dataset than allowing addition of a few elements to D which allows addition of some outliers which distorts too much.

#### Earth Mover Distance (Wasserstein Distance)
Quantifying distance between 2 distributions defined over a metric space.

Defined as the minimum transportation cost to move soil around to change the distribution landscape, where cost is per mass per distance moved.

**Post-Processing Theorem:** DP on the output of a DP mechanism does not result in any gain of privacy.

#### Robust Privacy
Two mechanisms are (\epsilon, \delta) indistinguishable if they are close in Wasserstein distance.

They are only distinguishable if there is some utility in the distinguishability (i.e. in a useful way). 

Out of the box, Differential Privacy does not provide this guarantee.


Typically cryptography demands correctness of output, while DP allows leeway on the correctness of the output to provide for privacy.

### One Way Secure Computation
- OWSC captures the idea of secure computation using noisy channels.
- There exist finite channels that are OWSC-complete with inverse-polynomial error.
- Characterization of channels that allow ZK functionality.
- Open problem: Can we characterize channels that are OWSC-complete?
- Bit-ROT and String-ROT

### On Communication Models and Best-Achievable Security in 2-Round MPC
MPC: Realizing functionality of a trusted central party in real world, with untrusted adverserial parties.

Communication Models:
- Broadcast
- Point-to-Point
- Public Key Infrastructure

Adversary Models:
- Identifiable Abort
- Guaranteed Output Delivery

Transformation from 2-round n-party over BC with only one corruption into 2-message secure OT protocol.

Equivalence of dishonest and honest majority wrt OT.

If a corrupt party does not send a message, honest parties cannot abort since it is indistinguishable from the corrupt receiver which lies about not receiving the message.

#### Summary
- Reduction to OT in the broadcast-only model achieving full malicious security? (Presently only semi-honest secure)
- Improving communication and computational efficiencies of the current 2-round protocol.

### Group Correlations and Applications in Cryptography (Guru Vamsi Polacherla)
- Correlations in MPC

### Breaking Barriers in Secret Sharing (Vinod Vaikuntanathan)
Any authorized subset of parties can recover the secret, while no other subset of parties can recover any information about the secret.

- Threshold Secret Sharing (Shamir '79, Blakely '79), where authorized subset has size >= t
- General SS (Ito-Saito-Nishizeki '87), where tha authorized subsets are defined by a monotone function.
	- Access Structure (monotone since a subset which includes an allowed subset must also be allowed). Monotone means that if a set of parties is authorized then every superset is also authorized.

#### General Secret Sharing
For every monotone access structure on n parties, there is a secret-sharing scheme with share size of 2^{0.994n} bits.

Private Information Retrieval, Conditional Disclosure of Secrets

### Conditional Disclosure of Secrets
Public Fn F

Alice has x and Bob has y. Charlie should learn secret only if F(x,y) = 1, without communication with each other and only communication with Charlie.

Solution: Alice has a truth table.

Communication is exponential in n. Protocols have linear reconstruction functions. However, message sizes need to be atleast O(2^n).

To reduce message size, we use non-linear function.

#### Private Information Retrieval
To reduce communication smaller than the entire database (truth table).

3-party primitive, where Alice and Bob share a database and Charlie holds an index x. The goal is that neither Bob nor Alice must learn the index x, while Charlie gets the results.

##### PIR vs CDS
Privacy: n bits (index) vs single bit
Randomness: no common randomness vs common random string R
Database: Replication of database vs No replication (one has a database and other has an index)

##### Transformation of PIR upper bound to CDS
Query Charlie sends queries q_0 and q_1 to DB0 and DB1 and Response Charlie gets H(q_0) and H(q_1).

q_0 and q_1 can be looked at a secret sharing of the original query/secret.

Suppose the PIR is a linear reconstruction PIR. Query Charlie is actually Alice and the DB0 is Bob. The first message is a random string which is shared with Bob.

Response Charlie and DB1 are represented by Charlie, which knows the database in CDS and gets the response as well.

Unfortunately, privacy is not preserved. Bob's message reveals the one-time pad randomness. A quick fix is to again add a random string to Bob's message.

#### CDS and Secret Sharing
Multiparty CDS for a forbidden hypergraph where the number of access functions is 2^{2^n/2}

There are N parties which hold the index while one party holds the database.

2-party CDS can be generalized to multi-party CDS.

#### Secret Sharing
Slice functions: Hamming weight type strings

Represent general monotone function circuit as a combination of slice functions. Use every possible slice function as a basis to construct the general function.

#### Locally Decodable Codes

#### Private Simultaneous Messages

### Indistinguishability Obfuscation: New Opportunities (Amit Sahai)

#### Indistinguishability Obfuscation
Apply obfuscation compiler to circuits which compute the same function, output is hard to distinguish.

New Cryptography:
- Functional Encryption
- Witness Encryption
- (Doubly) Deniable encryptions
- Hardness of finding hash
- Correlation Interactable Hash etc.

iO research: far from practical applications, exponential bounds

**Whitebox cryptography** - ???

### Format Preserving Encryption
Block cipher: Family of permutations indexed by the secret key.

Changing secret key in block cipher is quite costly.

Liskov, Rivest and Wagner (Crypto 2002) - define a tweak as a third input.

Brightwell and Smith, 1997 - Cipher text to plaintext relation

FPE scheme applications include privacy of SSN and database entries.

User specifies a format, the ciphertext must be within the same domain as the plaintext.

#### FPE Security Notions (BRRS'09) - Bellare, Rogaway, Stegers
- PRP: 
	- Overkill in terms of performance (not efficient to handle PRP security)
- SPI: Single Point Indistinguishability
	- Distinguishing encryption of specific message m from that of a random string r
- Message Privacy:
	- Encryption reveals no information except format
- Message Recovery:
	- Cannot completely reveal message from the encrypted ciphertext.

#### FPE techniques by Black and Rogaway
- Rank-then-Encipher approach
- Cyclic Technique
	- Keep encrypting until ciphertext lies within same set.
- Generalized Feistel


Specific implementations use Feistel type structure and are FF1, FF2 and FF3 and FEA1/FEA2.
FF1 - FF3 have been broken with known attacks while Korean standards FEA1/FEA2 have no known attacks.

#### Substitution Permutation Network
FPE designed using SPN paradigm


### Authenticated Encryption

### Lightweight Cryptography
Solutions for resource-constrained devices such as RFID tags, smart cards, sensor nodes etc.

### Oblivious and Fair Data Trading on Blockchain
Privacy in data owner and user transactions, where data owner does not know what user requests.

Owner has multiple files where each file has a searchword. User has searchword and money. User wants to purchase file containing searchword from the owner.

#### Oblivious Search
User can obliviously search for a document, located in third party remote server or in owner's local database

Goal: Privacy of searchword

Crypto: Searchable Encryption Scheme

ODT algorithm using Bilinear Maps


## Useful General Links
1. [Ben Lynn's set of assorted notes](https://crypto.stanford.edu/pbc/notes/)
2. [Set of relevant crypto papers](https://eprint.iacr.org/complete/)
