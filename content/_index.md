---
title: Security Analysis of Telegram (Symmetric Part)
draft: false
---

# Overview

We performed a detailed security analysis of the encryption offered by the popular [Telegram](https://telegram.org/) messaging platform. As a result of our analysis, we found several cryptographic weaknesses in the protocol, from technically trivial and easy to exploit to more advanced and of theoretical interest. 

For most users, the immediate risk is low, but these vulnerabilities highlight that Telegram fell short of the cryptographic guarantees enjoyed by other widely deployed cryptographic protocols such as TLS. We made several suggestions to the Telegram developers that enable providing formal assurances that rule out a large class of cryptographic attacks, similarly to other, more established, cryptographic protocols.

By default, Telegram uses its bespoke [MTProto](https://core.telegram.org/mtproto) protocol to secure communication between clients and its servers as a replacement for the industry-standard [Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) (TLS) protocol. While Telegram is often referred to as an "encrypted messenger", this level of protection is the only protection offered by default: MTProto-based end-to-end encryption, which would protect communication from Telegram employees or anyone breaking into Telegram's servers, is only optional and not available for group chats. 

We thus focused our efforts on analysing whether Telegram's MTProto offers comparable privacy to surfing the web with HTTPS.

# Vulnerabilities

We disclosed the following vulnerabilities to the Telegram development team on 16 April 2021 and agreed with them on a disclosure on 16 July 2021:

1. An attacker on the network can reorder messages coming from a client to the server. This allows, for example, to alter the order of "pizza" and "crime" in the sequence of messages: "I say yes to", "all the pizzas", "I say no to", "all the crimes". This attack is trivial to carry out. Telegram confirmed the behaviour we observed and addressed this issue in version 7.8.1 for Android, 7.8.3 for iOS and 2.8.8 for Telegram Desktop.

2. An attacker can detect which of two special messages was encrypted by a client or a server under some special conditions. In particular, Telegram encrypts acknowledgement messages, i.e. messages that encode that a previous message was indeed received, but the way it handles the re-sending of unacknowledged messages leaks whether such an acknowledgement was sent and received. This attack is mostly of theoretical interest. However, cryptographic protocols are expected to rule out even such attacks. Telegram confirmed the behaviour we observed and addressed this issue in version 7.8.1 for Android, 7.8.3 for iOS and 2.8.8 for Telegram Desktop.

3. We also studied the implementation of Telegram clients and found that three of them (Android, iOS, Desktop) contained code which -- in principle -- permitted to recover some plaintext from encrypted messages. For this, an attacker must send many carefully crafted messages to a target, on the order of millions of messages. This attack, if executed successfully, could be devastating for the confidentiality of Telegram messages. Luckily, it is almost impossible to carry out in practice. In particular, it is mostly mitigated by the coincidence that certain metadata in Telegram is chosen randomly and kept secret. The presence of these implementation weaknesses, however, highlights the brittleness of the MTProto protocol: it mandates that certain steps are done in a problematic order (see discussion below), which puts significant burden on developers (including developers of third-party clients) who have to avoid accidental leakage. The three official Telegram clients which exhibit non-ideal behaviour are evidence that this is a high burden. Telegram confirmed the attacks and rolled out fixes to all three affected clients in June. Telegram also awarded a "bug bounty" for these vulnerabilities.

4. We also show how an attacker can mount an "[attacker-in-the-middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)" attack on the initial key negotiation between the client and the server. This allows an attacker to impersonate the server to the client, allowing to break confidentiality and integrity of the communication. Luckily, this attack is also quite difficult to carry out, as it requires sending billions of messages to a Telegram server within minutes. However, it highlights that while users are required to trust Telegram’s servers, the security of those servers and their implementations cannot be taken for granted. Telegram confirmed the behaviour and implemented some server-side mitigations. In addition, from version 7.8.1 for Android, 7.8.3 for iOS and 2.8.8 for Telegram Desktop client apps support an [RSA-OAEP+](https://www.shoup.net/papers/oaep.pdf) variant.

We were informed by the Telegram developers that they do not do security or bugfix releases except for immediate post-release crash fixes. The development team also informed us that they did not wish to issue security advisories at the time of patching, nor commit to release dates for specific fixes. As a consequence, the fixes were rolled out as part of regular Telegram updates.

# Formal Security Analysis

The central result of our investigation, however, is that Telegram's MTProto *can* provide a confidential and integrity-protected channel when the changes we suggested are adopted by the Telegram developers. As mentioned above, the Telegram developers communicated to us that they did adopt these changes. Telegram awarded a cash price for this analysis to stimulate future analysis.

However, this result comes with significant caveats. Cryptographic protocols like MTProto are built from cryptographic building blocks such as hash functions, block ciphers and public-key encryption. In a formal security analysis, the security of the protocol is reduced to the security of its building blocks. This is no different to arguing that a car is road safe if its tires, brakes and indicator lights are fully functional. In the case of Telegram, the security requirements on the building blocks are unusual. Because of this, these requirements have not been studied in previous research. This is somewhat analogous to making assumptions about a car's brakes that have not been lab-tested. Other cryptographic protocols such as TLS do not have to rely on these sort of special assumptions.

A further caveat of these findings is that we only studied three official Telegram clients and no third-party clients. However, some of these third-party clients have substantial user bases. Here, the brittleness of the MTProto protocol is a cause for concern if the developers of these third-party clients are likely to make mistakes in implementing the protocol in a way that avoids, e.g. the timing leaks mentioned above. Alternative design choices for MTProto would have made the task significantly easier for the developers.

# Paper

Martin R. Albrecht, Lenka Mareková, Kenneth G. Paterson, Igors Stepanovs: [*Four Attacks and a Proof for Telegram*](/paper.pdf). To appear at [IEEE Symposium on Security and Privacy 2022](https://www.ieee-security.org/TC/SP2022/).

# Team

- [Martin R. Albrecht](https://malb.io) (Information Security Group, Royal Holloway, University of London)
- [Lenka Mareková](https://pure.royalholloway.ac.uk/portal/en/persons/lenka-marekova(371a2632-c1c4-4866-8141-c4b99807d326).html) (Information Security Group, Royal Holloway, University of London)
- [Kenneth G. Paterson](https://inf.ethz.ch/people/person-detail.paterson.html) (Applied Cryptography Group, ETH Zurich)
- [Igors Stepanovs](https://sites.google.com/site/igorsstepanovs/) (Applied Cryptography Group, ETH Zurich)

# A Somewhat Opinionated Discussion

“Don’t roll your own crypto” is a common mantra issued when a cryptographic vulnerability is found in some protocol. Indeed, Telegram has been the recipient of unsolicited advice of this nature. The problem with this mantra is, of course, that it sounds like little more than gatekeeping. Clearly, some people need to roll “their own crypto” for cryptography to be rolled at all.

However, despite the gatekeeping flavour, there is a rationale behind this advice. Standard cryptographic protocols have received attention from analysts and new protocols are developed in parallel with a proof that roughly says: “No adversary with these capabilities can break the given well-defined security goals unless one of the underlying primitives – a block cipher, a hash function etc – has a weakness.“ Of course, proofs can have bugs too, but this process significantly reduces the risk of catastrophic failure.

Two of our attacks described above serve to illustrate that some behaviours exhibited by Telegram clients and servers are undesirable (permitting reordering of some messages, encrypting twice under the same state in some corner case). The apparent need to make non-standard assumptions on the underlying building blocks in our proofs (which we do not know how to avoid) further illustrates that some design choices made in MTProto are more risky than they need to be. In other words, this part of our paper – “Two Attacks and a Proof” so to speak – illustrates the implied rationale of the above mentioned mantra: proofs help to reduce the attack surface.

But there is another leg of the “don’t roll your own crypto” mantra: it can be surprisingly tricky to implement cryptographic algorithms in a way that they leak no secret information, e.g. through timing side channels. Proofs only cover what is in their model. Two of our attacks are timing attacks and thus “outside the model”. Our proof essentially states that it is, in principle, possible to implement MTProto in a way that is secure but does not cover how easy or hard it is or how to do it at all. Here, two recurring "anti-patterns" in MTProto’s design make it tricky to implement the protocol securely.

First, MTProto opts to protect the integrity of plaintexts rather than ciphertexts. This is the difference between [Encrypt-and-MAC](https://en.wikipedia.org/wiki/Authenticated_encryption#Approaches_to_authenticated_encryption) and Encrypt-then-MAC. It might seem natural to protect the integrity of the part that you care about – the plaintext – but doing it this way around means that a receiver must first process an incoming ciphertext with their secret key (i.e. decrypt it) *before* being able to verify that the ciphertext has not been tampered with. In other words, the receiver must perform a computation involving untrusted data – the received ciphertext – and their decryption key. This can be done in a secure manner, but Encrypt-then-MAC completely sidesteps the issue by first checking whether the ciphertext was tampered with (i.e. checking the MAC on the ciphertext), and only then decrypting.

Second, but related to the first point, block ciphers process data in blocks of e.g. 16 bytes. Since data may have an arbitrary byte length, there will be some bytes left over that MTProto fills with random garbage (which is good). Now, since Telegram protects the integrity of plaintexts instead of ciphertexts, the question arises: compute the MAC over the plaintext with or without the padding? The original design decision was “without padding”, presumably because the designers did not see a need to protect useless random padding. The remaining two of our attacks exploit this behaviour.

1. As mentioned above, we break – in a completely impractical way! – the initial key exchange between the client and the server. Here, we exploit that MTProto attempts to add integrity inside RSA encryption by including a hash of the payload but excluding the added random padding. This is like a homegrown variant of [RSA-OAEP](https://en.wikipedia.org/wiki/Optimal_asymmetric_encryption_padding). The problem with this approach is that the receiver must – after decryption – figure out where the payload ends and where the padding starts. This means parsing the payload *before* being able to check its integrity. Furthermore, depending on the result of this parsing, more or less data may be fed into the hash function for integrity checking, which in turn produces slightly shorter or longer running times (our actual attack proceeds differently, we are merely illustrating the principle here while avoiding many details).

2. Our second attack goes for the one place where MTProto does indeed also protect useless random padding, but the processing in some clients behaves as if this was not the case. In 2015 Jakob Jakobsen and Claudio Orlandi [gave an attack](https://eprint.iacr.org/2015/1177) on the IND-CCA security of the previous version of MTProto. As a result of this, MTProto 2.0, the current version, now also protects the integrity of padding bytes. Thus, the logic now *could* be: (a) decrypt and then immediately (b) check integrity. (This, too, isn’t without its pitfalls. For example, when do you know that you have enough data to run the integrity check? Parsing some length field first, for example, to establish this [could again lead to attacks](https://en.wikipedia.org/wiki/Secure_Shell_Protocol#CBC_plaintext_recovery).) However, we found that three official Telegram clients do additional processing on the decrypted data before step (b), processing that is necessary in MTProto 1.0 (where padding and plaintext data needed to be separated before checking integrity) but not in MTProto 2.0 (where the integrity of everything is protected). We exploit – again in a completely impractical way! – this behaviour in our attacks (but we also need to combine it with the previous attack to make it all come together). So, again, the original decision not to protect some useless random bytes in MTProto 1.0 required the receiver to decide which bytes are useless and which aren’t *before* checking their integrity, and three official Telegram clients have carried this behaviour forward into MTProto 2.0.

   As an aside, Jakobsen and Orlandi wrote: “We stress that this is a theoretical attack on the definition of security and we do not see any way of turning the attack into a full plaintext-recovery attack.” Similarly, the Telegram “[FAQ for the Technically Inclined (MTProto v.1.0)]([https://web.archive.org/web/20210706124220/https://core.telegram.org/techfaq/mtproto_v1#what-about-ind-cca])” provides the following analogy: “A postal worker could write 'Haha' (using invisible ink!) on the outside of a sealed package that he delivers to you. It didn't stop the package from being delivered, it doesn't allow them to change the contents of the package, and it doesn't allow them to see what was inside.” In hindsight, we think that this is incorrect. As explained above, our timing side channels essentially exploit this behaviour in order to do message recovery (but we need to “chain” two “exploits” to make it work, even ignoring practicality concerns).

In summary, MTProto protects the integrity of plaintexts rather than ciphertexts, which necessitates operating with a decryption key on untrusted data. Moreover, MTProto in several places opted (or at least used to opt) to require additional parsing of decrypted data before its integrity could be checked by only protecting the payload without padding. This produces an opportunity for timing side-channel attacks; an opportunity that could be completely removed by using a standard authenticated encryption scheme (roughly speaking, Encrypt-then-MAC with key separation has been shown to be a decent such scheme, but faster dedicated schemes exist). Finally, given that Telegram’s ecosystem is serviced also by many third-party clients, the “brittleness” of the design or the presence of “footguns” means that even if the developers of the official clients manage to take great care to avoid timing leaks, those are difficult to rule out for third-party clients.

# Q & A

## What about IGE?

Telegram uses the little-known Infinite Garble Extension (IGE) block cipher mode in place of more standard alternatives. While claims about its infinite error propagation have been [disproven](https://groups.google.com/forum/#!topic/sci.crypt/4bkzm_n7UGA), our proofs show that its use in the symmetric part of MTProto is no more problematic than if CBC mode was used. However, its similarity with CBC also means it is vulnerable to manipulation if some bits of plaintext are known. Indeed, we use this property in combination with the timing side channel described earlier.

## What about length extension attacks?

MTProto makes heavy use of plain SHA-256, both in deriving keys and calculating the MAC, which on first look appears as the kind of use that would lead to [length extension attacks](https://en.wikipedia.org/wiki/Length_extension_attack). However, as our proofs show, MTProto manages to sidestep this particular issue because of its plaintext encoding format which mandates the presence of certain metadata in the first block.

## Did we really break IND-CPA?

Above, we wrote:  
> An attacker can detect which of two special messages was encrypted by a client or a server under some special conditions. In particular, Telegram encrypts acknowledgement messages, i.e. messages that encode that a previous message was indeed received, but the way it handles the re-sending of unacknowledged messages leaks whether such an acknowledgement was sent and received. This attack is mostly of theoretical interest. However, cryptographic protocols are expected to rule out even such attacks. Telegram confirmed the behaviour we observed and addressed this issue in version 7.8.1 for Android, 7.8.3 for iOS and 2.8.8 for Telegram Desktop.

Telegram [wrote](https://web.archive.org/web/20210716150303/https://core.telegram.org/techfaq/UoL-ETH-4a-proof#2-implications-for-ind-cpa-security-sections-iv-b2-iv-c7):  
> MTProto never produces the same ciphertext, even for messages with identical content, because MTProto is stateful and msg_id is changed on every encryption. If one message is re-sent on the order of 2^64 times, MTProto can transmit the same ciphertext for the same message on two of these re-sendings. However, it would not be correct to claim that retransmission over the network of the same ciphertext for a message that was previously sent is a violation of IND-CPA security because otherwise any protocol over TCP wouldn't be IND-CPA secure due to TCP retransmissions.
> 
> To facilitate future research, each message that is re-sent by Telegram apps is now either wrapped in a new container or re-sent with a new msg_id.

We have already addressed this in the latest version of our write-up (which we shared with the Telegram developers on 15 July 2021). We reproduce that part below, slightly edited for readability.

> If a message is not acknowledged within a certain time in MTProto, it is re-encrypted using the same `msg_id` and with fresh random padding. While this appears to be a useful feature and a mitigation against message deletion, it enables attacks in the IND-CPA setting, as we explain next.

> As a motivation, consider a local passive adversary that tries to establish whether `R` responded to `I` when looking at a transcript of three ciphertexts (`c_{I, 0}`, `c_{R}`, `c_{I, 1}`), where `c_{u}` represents a ciphertext sent from `u`. In particular, it aims to establish whether `c_{R}` encrypts an automatically generated acknowledgement, we will use “`\ACK`” below to denote this, or a new message from `R`. If `c_{I, 1}` is a re-encryption of the same message as `c_{I, 0}`, re-using the state, this leaks that bit of information about `c_{R}`.

> Note that here we are breaking the confidentiality of the ciphertext carrying “`ACK`”. In addition to these encrypted acknowledgement messages, the underlying transport layer, e.g. TCP, may also issue unencrypted ACK messages or may resend ciphertexts as is. The difference between these two cases is that in the former case the acknowledgement message is encrypted, in the latter it is not. For completeness, note that Telegram clients do not resend cached ciphertext blobs when unacknowledged, but re-encrypt the underlying message under the same state but with fresh random padding.

These pararagraphs are then followed by a semi-formal write-up of the attack.