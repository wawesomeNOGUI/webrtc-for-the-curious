---
title: Securing
type: docs
weight: 5
---


## What security does WebRTC have?
Every WebRTC connection is authenticated and encrypted. You can be confident that a 3rd party can't see what you are sending. They also can't insert bogus messages. You can also be sure the WebRTC Agent that generated the Session Description is the one you are communicating with.

It is very important that no one tampers with those messages. It is ok if a 3rd party reads the Session Description in transit. However WebRTC has no protection against it being modified. An attacker could MITM you by changing ICE Candidates and the Certificate Fingerprint.


## How does it work?
WebRTC uses two pre-existing protocols [DTLS](https://tools.ietf.org/html/rfc6347) and [SRTP](https://tools.ietf.org/html/rfc3711)

DTLS allows you negotiate a session and then exchange data securely between two peers. It is a sibling of TLS, the same technology that powers HTTPS. DTLS is over UDP instead of TCP, so the protocol has to handle unreliable delivery. SRTP is designed just for exchanging media. There are some optimizations we can make by using it instead of DTLS.

DTLS is used first. It does a handshake over the connection provided by ICE. DTLS is a client/server protocol, so one side needs to start the handshake. The Client/Server roles are chosen during signaling. During the DTLS handshake both sides offer a certificate.
After the handshake is complete this certificate is compared to the certificate hash in the Session Description. This is to assert that the handshake happened with the WebRTC Agent you expected. The DTLS connection is then available to be used for DataChannel communication.

To create a SRTP session we initialize it from the keys generated by DTLS. SRTP does not have a handshake mechanism, so has to be bootstrapped with external keys. When this is done media can be exchanged that is encrypted via SRTP!

## Security 101
To understand the technology presented in this chapter you will need to understand these terms first. Cryptography is a tricky subject so would be worth consulting other sources also!

#### Cipher
Cipher is a series of steps that takes plaintext to ciphertext. The cipher then can be reversed so you can take your ciphertext back to plaintext. A Cipher usually has a key to change its behavior. Another term for this is encrypting and decrypting.

A simple cipher is ROT13. Each letter is moved 13 characters forward. To undo the cipher you move 13 characters backwards. The plaintext `HELLO` would become the ciphertext `URYYB`. In this case the Cipher is ROT, and the key is 13.

#### Plaintext/Ciphertext
Plaintext is the input to a cipher. Ciphertext is the output of a cipher.

#### Hash
Hash is a one way process that generates a digest. Given an input it generates the same output every time. The output is not reversible, a hash input shouldn't be guessable from an input. Hashing is useful when you want to confirm that a message hasn't been tampered.

A simple hash would be to only take every other letter `HELLO` would become `HLO`. You can't assume `HELLO` was the input, but you can confirm that `HELLO` would be a match.

#### Public/Private Key Cryptography
Public/Private Key Cryptography describes the type of ciphers that DTLS and SRTP uses. In this system you have two keys, a public and private key. The public key is for encrypting messages, and is is safe to share.
The private key is for decrypting, and shouldn't be shared. It is the only key that can decrypt the messages encrypted with the public key.

#### Diffie–Hellman exchange
Diffie–Hellman exchange allows two users who have never met before to share a value securely over the internet. User `A` can send a value to User `B` without worrying about eavesdropping. This depends on the difficulty of breaking the discrete logarithm problem.
You don't have to fully understand this, but it helps to understand how the DTLS handshake is possible.

Wikipedia has an example of this in action [here](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange#Cryptographic_explanation)

## DTLS
DTLS (Datagram Transport Layer Security) allows two peers to establish secure communication with no pre-existing configuration. Even if someone is eavesdropping on the conversation they will not be able to decrypt the messages.

For A DTLS Client and Server to communicate they need to agree on a cipher and the key. They determine these values by doing a DTLS handshake. During the handshake the messages are in plaintext.
When a DTLS Client/Server has exchanged enough details to start encrypting it sends a `Change Cipher Spec`.  After this message each subsequent message is encrypted!

### Packet Format
Every DTLS packet starts with a header

#### Content Type
You can expect the following types

* Change Cipher Spec - `20`
* Handshake - `22`
* Application Data - `23`

`Handshake` is used to exchange the details to start the session. `Change Cipher Spec` is used to notify the other side that everything will be encrypted. `Application Data` are the encrypted messages.

#### Version
Version can either be `0x0000feff` (DTLS v1.0) or `0x0000fefd` (DTLS v1.2) there is no v1.1

#### Epoch
The epoch starts at `0`, but becomes `1` after a `Change Cipher Spec`. Any messages with a non-zero epoch is encrypted.

#### Sequence Number
Sequence Number is used to keep messages in order. Each message increases the Sequence Number. When the epoch is incremented the Sequence Number starts over.

#### Length and Payload
The Payload is `Content Type` specific. For a `Application Data` the `Payload` is the encrypted data. For `Handshake` it will be different depending on the message.

The length is for how big the `Payload` is.

### Handshake State Machine
During the handshake the Client/Server exchange a series of messages. These messages are grouped into a flights. Each flight may have multiple messages in it (or just one).
A Flight is not complete until all message in the flight have been received. We will describe the purpose of each message in greater detail below,

{{<mermaid>}}
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: ClientHello
    Note over C,S: Flight 1

    S->>C: HelloVerifyRequest
    Note over C,S: Flight 2

    C->>S: ClientHello
    Note over C,S: Flight 3

    S->>C: ServerHello
    S->>C: Certificate
    S->>C: ServerKeyExchange
    S->>C: CertificateRequest
    S->>C: ServerHelloDone
    Note over C,S: Flight 4

    C->>S: Certificate
    C->>S: ClientKeyExchange
    C->>S: CertificateVerify
    C->>S: ChangeCipherSpec
    C->>S: Finished
    Note over C,S: Flight 5

    S->>C: ChangeCipherSpec
    S->>C: Finished
    Note over C,S: Flight 6
{{< /mermaid >}}

#### ClientHello
ClientHello is the initial message sent by the client. It tells the server all the ciphers and features the client supports. It also contains random data that will be used to to generate the keys for the session.

#### HelloVerifyRequest
HelloVerifyRequest is sent by the server to the client. It is to make sure that the client intended to send the request. The Client then re-sends the ClientHello, but with a token provided in the HelloVerifyRequest.

#### ServerHello
ServerHello is the response by the server for the configuration of this session. It contains what cipher will be used when this session is over. It also contains the server random data.

#### Certificate
Certificate contains the certificate for the Client or Server. This is used to uniquely identify who we were communicating with. After the handshake is over we will make sure this certificate when hashed matches the fingerprint in the `SessionDescription`

#### ServerKeyExchange
ServerKeyExchange contains the server's public key.

#### CertificateRequest
CertificateRequest is sent by the server notifying the client that it wants a certificate. The server can either Request or Require a certificate.

#### ServerHelloDone
ServerHelloDone notifies the client that the server is done with the handshake

#### ClientKeyExchange
ClientKeyExchange contains the client's public key.

#### CertificateVerify
CertificateVerify is how the sender proves that it has the private key sent in the Certificate message.

#### ChangeCipherSpec
ChangeCipherSpec informs the receiver that everything sent after this message will be encrypted.

#### Finished
Finished is encrypted and contains a hash of all messages. This is to assert that the handshake was not tampered with.

## SRTP

## DTLS and SRTP together