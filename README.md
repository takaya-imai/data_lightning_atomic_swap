# Proposal: Data Lightning Atomic Swap (DLAS-down, DLAS-up)


# Abstract

This proposal is a way to swap data and lightning payment atomically.
It has two patterns, one is for a payer to swap data-download with lightning payment to a payee (DLAS-down), the other is for a payer to swap data-upload with lightning payment to a payee (DLAS-up).

The data is embedded to preimage so sending and receiving the data need lightning payment at the same time.


# Motivation

Atomic Swaps among crypto currencies has various ways to implement (on-chain to on-chain[1], on-chain to of-chain(Submarine Swap[2])). And Atomic Swaps between data and crypto currencies are also proposed as a part of TumbleBit mechanism[3], Storm mechanism[4] and so on.

Recently Joost Jager proposed Instant messages with lightning onion routing, whatsat[5], which use recent sphinx payload change[6]. This is very awesome but not atomic with lightning payment.

Atomic lightning mechanism for data is useful in use cases below.


# Pros & Cons

- DLAS-down
    - Pros
        - Atomic data download exchange with lightning payment
    -  Cons
        - It needs better mechanism to expand data size

- DLAS-up
    -  Pros
        - Atomic data upload exchange with lightning payment
    -  Cons
        - OG AMP[7] is needed to implement


# What I describe

- A way to swap data with lightning payment atomically.


# What I do not describe

- A way to detect that data is correct or not, namely zero knowledge proof process.

For example, probabilistic checkable proof like TumbleBit[3] proposed.
Just message as data is no problem because no need to check the message is correct or not. 

- A way in case that different preimages are used in a payment route like Multi-hop locks.


# Specification

Lightning Network(LN) has a mechanism about preimage like a brief image below. 


    Payer                             Mediators                            Payee
    =================================================================================
                                                                            Preimage
    Preimage Hash  <--------------------- invoice ------------------------  Preimage Hash
    Preimage Hash  ---------------->   Preimage Hash -------------------->  Preimage Hash
    Preimage       <—-------------—-   Preimage      <--------------------  Preimage

As you know, preimage Payer gets can be a proof of payment because Payer can not get it if the payment is executed correctly.



1. Data download <->  lightning (DLAS-down)


Payer sends lightning payment and receives data from Payee atomically.


    Payer                             Mediators                           Payee
    =================================================================================
    Payer Channel Pubkey <-----------------------------------------------> Payee Channel Pubkey
    
                                                                           data(256bit, padded)
                                                                           enc_key = (Payee Channel Secret Key * Payer Channel Pubkey).x  (256bit)
    enc_key = (Payer Channel Secret Key * Payee Channel Pubkey).x  (256bit)
                                                                           enc_data = data XOR enc_key
    sha256(enc_data) <--------------------- invoice ---------------------- sha256(enc_data)
    sha256(enc_data) ----------------> sha256(enc_data) -----------------> sha256(enc_data)
    enc_data         <---------------- enc_data <------------------------- enc_data
    data = enc_data XOR enc_key


- The size of data is restricted to 256 bits. Identically, it should be extended to larger data and the data should be transferred in several payment paths like DLAS-up.
- Channel Pubkey is only one for one channel and the data can be decrypted if enc_key is leaked. So enc_key should be generated newly every time by a way like hash chain but the protocol image above is just example for simplicity.
- .x means X axis value of points on Elliptic Curve.
- If data is less than 256 bits, then 0x00 is padded (I am not sure which of big endian and little endian is better).



2. Data upload <->  lightning (DLAS-down)

Payer sends data and lightning payment from Payee atomically.
This is like OG AMP(Atomic Multi-path Payment)[7] system.


    Payer                             Mediators                            Payee
    =================================================================================
    data(512bit, padded)
    
    share1(256bit)
    share2(256bit)
    
    base_s = share1 XOR share2
    data1(256bit) ||  data2(256bit) = data(512bit)
    XOR_d1 = data1 XOR base_s
    XOR_d2 = data2 XOR base_s
    PreImg1 = sha256(base_s || data || 1)
    PreImg2 = sha256(base_s || data || 2)
    
    sha256(PreImg1), XOR_d1, share1 -> sha256(PreImg1), XOR_d1, share1  -> sha256(PreImg1), XOR_d1, share1
    sha256(PreImg2), XOR_d2, share2 -> sha256(PreImg2), XOR_d2, share2  -> sha256(PreImg2), XOR_d2, share1
    
                                                                           base s = share1 XOR share2
                                                                           data = (XOR_d1 XOR base_s) || (XOR_d2 XOR base_s)
                                                                           PreImg1 = sha256(base_s || data || 1)
                                                                           PreImg2 = sha256(base_s || data || 2)
    
    PreImg1    <-------------------    PreImg1    <---------------------   PreImg1
    PreImg2    <-------------------    PreImg2    <---------------------   PreImg2


- This protocol example has 512 bits data and they are transferred in two paths. However, it can transfer larger data in several payment paths like [5].
- || means string concatenation.
- If data is less than 512 bits, then 0x00 is padded(I am not sure which of big endian and little endian is better).




# Use Cases

1. Lightning Network ecosystem

- Hosting Incentives like Acai Protocol
    - Watchtower Hosting incentive, Backup Hosting incentive
        - Commitment tx data sending to Data Host(DLAS-up)
            - Commitment tx data is embedded in preimage so that Payer can not send the data without remittance
        - Channel backup data receiving from Data Host(DLAS-down)
            - Channel backup data is embedded in preimage so that Payer can not receive the data without remittance

2. Crypto currency Problems

- Distributed secret key sharing (just come up with an idea though)
    - As a key backup, one of secret key shares is distributed with encryption(DLAS-up) to some nodes, which nodes receive lightning payment as key managing fee. And the nodes send a proof for managing the key as response of bloom filter periodically, and exchange encrypted secret key share with lightning payment to asset holder(DLAS-down).
    - For example 2 out of 3 multi signature key sharing, asset holder puts the first key, the custodial has the second key, and the third key at the lightning distribution nodes. Asset holders usually spend assets using their key and the key on Distributed Nodes.


3. Problems so far

- Prevention email spam and DDoS attack with large data
    - Payer can not send email or data without remittance(DLAS-up)
    - Payer can not receive reply-email without remittance(DLAS-down)

- Incentive of receiving advertisements on browser or desktop/mobile app
    - Payer can not send advertisements without remittance(DLAS-up)

- Bounty for code bug fixes based on cryptographic proofs or secret computations
    - DLAS-down



# References

- [1] https://bitcointalk.org/index.php?topic=321228
- [2] https://twitter.com/roasbeef/status/964608261830750208
- [3] https://eprint.iacr.org/2016/575
- [4] https://github.com/storm-org/storm-spec
- [5] https://twitter.com/joostjgr/status/1190714028626251779
- [6] https://github.com/lightningnetwork/lightning-rfc/pull/619
- [7] https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-February/000993.html
