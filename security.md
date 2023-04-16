# Security Modelling Cloak

## Introduction

Cloak is designed to be a secure layer on top of the messaging app. It is NOT a messaging app unto itself.You entrusted Cloak with all your messages and secrets. For this reason, we take security very seriously. This document describes the security model of Cloak and the security guarantees it provides.

### Security Model

The first steps in defining a security model are to identify the assets, the attack surface, and the motivation of the attacker. In our case, the assets are the messages that are being sent over the underlying messaging app. The 'attacker' in our case, is the messaging app itself, and the backdoors they have implemented behind your back. Their main motivation is to read and identify patterns in your messages. The attack surface they have to work with is the communication between Cloak and the messaging app, as well as the at-rest storage of the messages on their server.

### Other Considerations

To ensure smooth functioning as a lightweight layer on top of the messaging app, Cloak also tries to be as lean as possible and doesn't implement any extra security feature that is not relevant to the threat model.

## How we secure your messages

<img src="https://i.imgur.com/2l2Lduf.png" style="filter: invert(1)">

<p style="text-align: center"> <em>Where cloak fits in</em> </p>

Cloak uses Elliptic curve diffie hellman to establish a secure channel. Now that we have established a secure channel, let us look at what exactly we want out of our encryption scheme, and what we don't need.

### What we need

-   A strong encryption algorithm that provides confidentiality
-   Ephemeral keys to prevent long term key compromise
-   Forward secrecy to prevent compromise of past messages
-   Protection against crib dragging, and other known plaintext attacks

### What we don't need

-   Integrity
-   Authentication

### Why don't we need integrity and authentication?

As we discussed before, the main motive of our attacker here is to read and identify patterns in your messages. They are not interested in modifying your messages. They are only interested in reading your messages. So, it is reasonable to assume that the messaging app will provide these services without fail. This is why we don't need integrity and authentication.

### Why do we need forward secrecy?

Forward secrecy is a property of encryption schemes that prevents compromise of past messages in case of long term key compromise. This is important because the messaging app can compromise your long term key at any time. This is why we need forward secrecy.

### Why do we need ephemeral keys?

Ephemeral keys are keys that are used only once. They are generated on the fly and are never stored. This is important because the messaging app can compromise your long term key at any time. This is why we need ephemeral keys.

### Why do we need protection against crib dragging?

Crib dragging is a known plaintext attack where the attacker can decrypt a message by using a known plaintext. This is important as it can be used to identify patterns in your messages. This is why we need protection against crib dragging.

## How we implement these security features

### Encryption

We don't use AES in ECB mode. We use AES in CTR mode. This is because AES in ECB mode is vulnerable to known plaintext attacks, such as crib dragging. To mitigate this, we need to use a mode of operation that uses a nonce i.e a different key for each message. This is why we use AES in CTR mode.

#### Why don't we use AES in GCM mode?

AES in GCM mode, while being the industry standard for authenticated encryption, is not necessary for our purpose, and the added overhead over CTR mode is not worth it. We don't need authentication or integrity as discussed before, and those are only things we gain from using GCM mode.

### Decryption

AES in CTR mode has the same function for encryption and decryption, and thus shares the same properties.

## How exactly does the trade-off between modes of AES work?

Assume for the discussion below, that E<sub>k</sub> is a function that encrypts a message using the AES routine, with the key K.

### AES in ECB mode

In this mode, if a message `m1` and `m2` sent at different times, but having the same contents, will result in the ciphertexts `c1` and `c2` being the same as well. This allows the backdoor to garner patterns from the messages, which is not something that is desirable.

### AES in CTR Mode

In this mode, we fix this issue by performing the following operation

We will take our initialization vector (IV) and append to it our counter value, encrypt it before XORing it with our plaintext.

$$
c_1 = E_k(IV\ ||\ CTR[1])\ \oplus\ m_1
$$

and

$$
c_2 = E_k(IV\ ||\ CTR[2])\ \oplus\ m_2
$$

In this scheme, we use a different key for every message, and thus even if `m1` and `m2` are identical, their ciphertexts will be different.

## Steps taken to prevent crib dragging

Crib dragging refers to an attack which can occur when a OTP (the `IV || CTR` in our case) is used incorrectly, and we end up using the same OTP for two different messages.

Then, the attacker (the backdoor) can make use of the fact that

$$
c_1 \oplus c_2 = E_k(IV\ ||\ CTR[1])\ \oplus\ m_1 \oplus E_k(IV\ ||\ CTR[2])\ \oplus\ m_2
$$

Now, if the counters were incorrectly used, and they ended up being the same, then $E_k$ will return the same as well, meaning the XOR of them will cancel out!

This means

$c_1 \oplus c_2 = m1 \oplus m2$

Now, if the attacker can guess a word that is part of `m1`, maybe it's 'journalist', then dragging the word across $c_1 \oplus c_2$ window by window, will give the corresponding word in `m2`. (Same holds vice versa).

They will keep trying each window until they get a word that makes sense in the context of the vocabulary being used.

Then, the extracted words can be used again and again to slowly reveal the entire message.

To prevent this, cloak uses the stable RNG `crypto.getRandomValues()` to generate the IV. The AES function provided in the `WebCrypto` API also uses a stable, thread-safe `counter` value to ensure that no two keys are the same.

### Worked out example of crib dragging

Assume that

-   message1 = `Hello World`
-   message2 = `the program`
-   vulnerable key = `0x7375706572736563726574`

meaning the ciphertext $c_1$ = `0x3b101c091d53320c000910`

and $c_2$ = `0x071d154502010a04000419`

$c_1 \oplus c_2$ = `0x3c0d094c1f523808000d09`

Let us guess `the` as the crib word.

trying out each window, we can see that in the window given below,

```
   3c0d094c1f523808000d09
XOR 746865
---------------------------
    48656c
```

Now, this value results in `Hel` when converted to ASCII.

Next, we guess that this is likely `Hello` and perform crib dragging with `Hello` again and guess more and more of the plaintexts, finally resulting in the breaking of both messages.

### Mitigation

As mentione earlier, Cloak mitigates this by ensuring that key-reuse never happens.

In that case, the equality the above attack rests upon viz. $c_1 \oplus c_2$ = $m_1 \oplus m_2$ doesn't hold and the attacker is helpless, as the $E_k$ values don't cancel out in the XOR as they are derived from different nonces i.e
`IV || CTR[1]` and `IV || CTR[2]` where the two counters are necessarily distinct values.
