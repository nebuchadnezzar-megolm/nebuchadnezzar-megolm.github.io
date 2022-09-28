---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

title: Nebuchadnezzar
toc: true
---

# Overview

We report several practically-exploitable cryptographic vulnerabilities in the end-to-end encryption in [Matrix](https://en.wikipedia.org/wiki/Matrix_(protocol)) and describe proof-of-concept attacks exploiting these vulnerabilities. When relying on implementation specific behaviour, these attacks target the Matrix standard as implemented by the `matrix-react-sdk` and `matrix-js-sdk` libraries.[^1] These libraries provide the basis for the flagship [Element](https://element.io/) client. The vulnerabilities we exploit differ in their nature (insecure by design, protocol confusion, lack of domain separation, implementation bugs) and are distributed broadly across the different subprotocols and libraries that make up the cryptographic core of Matrix.

We target the setting where encrypted messaging and verification are enabled, i.e. in the presence of the strongest protections offered by the protocol. Furthermore, all attacks require cooperation of the homeserver. This is a natural threat model to consider, given that end-to-end encryption aims to provide protections against such untrusted third parties. We report the following vulnerabilities and attacks:

## Simple confidentiality break

Our first two attacks exploit the homeserver’s control over the list of users and/or devices in a room. Neither of these attacks break the confidentiality of Megolm or Olm protocols (see below) directly, instead they utilise functionality that puts too much trust in the homeserver to act honestly (in the context of end-to-end encryption, such trust should be minimised).

In the first attack, a malicious homeserver can add users under their control to end-to-end encrypted rooms. Once the homeserver-controlled user has been added, they may decrypt future messages sent in the room.

In the Matrix specification, existing users in a room (with sufficient permissions) may invite other users to join. To do this, they ask the homeserver to invite the user, which will in turn generate a room membership message. Then, to join the room, the invited user accepts the invite through the homeserver.

However, these room management messages are not authenticated in the Matrix specification (even for end-to-end encrypted rooms) and such an exchange of messages can be faked by a homeserver, essentially allowing a user under the homeserver's control to join a room (without the permission of the room's existing users).

Megolm, the central object in Matrix' secure messaging, is a group messaging protocol used to encrypt and decrypt messages sent by users. When a Megolm session is first started, the owner of the session (the device that will be sending messages using it) generates an outbound and an inbound session. The inbound session is used by other devices to decrypt and authenticate messages encrypted using the outbound session (kept by the session owner). The inbound session is distributed through an Olm channel (a pairwise encrypted messaging protocol). When a new user joins an end-to-end encrypted room, the existing members will share their inbound Megolm sessions through the Olm protocol when the next send a message. These allow the new member to decrypt future messages sent in the room. Thus, this first attack breaks the confidentiality of user messages.

While the Matrix specification does not require a mitigation of this behaviour, when a user is added to a room, Element will display this as an event in the timeline. Thus, to users of Element this is detectable. However, such a detection requires careful manual membership list inspection from users and to participants, this event appears as a legitimate group membership event. In particular, in sufficiently big rooms such an event is likely to go unnoticed by users.

In the second attack, a malicious homeserver adds a device under their control to another user's account in the room they wish to eavesdrop in. This device will be displayed and labelled as 'unverified' to all users in the room (and a warning icon will be added to the room and messages sent by the malicious device). Nonetheless, existing devices will share their inbound Megolm sessions with the new device, which allows decryption of future messages sent in the room. Thus, this second attack also breaks the confidentiality of user messages.

These are design issues with the Matrix specification and thus all clients are affected unless they implement additional countermeasures.

We discuss this attack in the section "Homeserver Control of Room Membership" of our research paper.

## Attack against out-of-band verification

Out-of-band verification in Matrix allows users to verify that the cryptographic identity they are communicating with matches the person they intend. Each user has a cross-signing identity that serves as the root-of-trust for their cryptographic identity. When completing the out-of-band verification process, the two devices establish a secure channel that they verify has not been interfered with, then share their cryptographic identities with each other. When each device receives the other's cryptographic identity, they sign it with their own. These signatures serve as a record of the out-of-band verification process, attesting to each device that the cryptographic identity they are interacting with matches the person it claims to.

However, in some areas of the Matrix specification, device identifiers and key identifiers (used to identify cryptographic identities) are used interchangeably.
In our attack, a malicious homeserver uses this lack of domain separation to convince their target to cryptographically sign (and thus verify) a cross-signing identity controlled under the homeserver's control. 
Once completed, this enables a [mallory-in-the-middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) (MITM) attack, breaking the confidentiality and authenticity of the underlying Olm channels (and thus also Megolm channels).

This issue derives from an insecure implementation choice permitted by the specification (which does not enforce domain separation between device and key identifiers). Today's disclosure fixes the insecure implementation choice, and the developers plan to separate the two formats in the specification at a later date.

We discuss this attack in the section "Key/Device Identifier Confusion in SAS" of our research paper.

## Semi-trusted impersonation

There exist situations where a device in Matrix is missing inbound Megolm sessions that it should have access to. For example, when a user adds a new device, that device should be able to decrypt messages sent before the device had been added. Since the device missed the initial distribution of such keys, it can use the 'Key Request protocol' to request a copy from other devices.

Matrix does not provide a cryptographic mechanism to ensure that the keys shared through the 'Key Request protocol' are legitimate. The specification therefore requires that sharing of inbound Megolm sessions should only be completed between devices that trust each other. For example, the reference implementation will only request missing keys from a user's own verified devices, or from the device that originally generated the session. As a further precaution, a warning message is added next to messages that have been decrypted using forwarded key.

Whilst Element clients restrict who they share keys with, no verification is implemented on who to accept key shares from. Our attack exploits this lack of verification in order to send attacker controlled Megolm sessions to a target device, claiming they belong to a session of the device they wish to impersonate. The attacker can then send messages to the target device using these sessions, which will authenticate the messages as coming from the device being impersonated. Whilst these messages will be accompanied by a warning, this is the same warning that accompanies keys *honestly* forwarded with the "Key Request protocol".

This is mostly implementation bug supported by a lack of guidance on the processing of incoming key shares in spec.

We discuss this attack in the section "Semi-trusted Impersonation against Megolm
Authentication" of our research paper.

## Trusted impersonation

When a Megolm session is first started, the owner of the session generates an outbound and an inbound session. The inbound session is distributed through an Olm channel. However, prior to today's fixes, `matrix-js-sdk` only required that such messages are encrypted, not that they were encrypted with an Olm channel. This opens the flagship client (and others utilising the `matrix-js-sdk` library) to a protocol confusion attack.

To start, the adversary establishes a Megolm channel using the previously mentioned semi-trusted impersonation attack (the *outer* session). They then initiate a second, *inner* Megolm session that is then distributed to other devices using a Megolm key share message (not as a forwarded room key). This *inner* shared Megolm session inherits its *sender* from the outer, forged Megolm session, but without inheriting its *forwarded* status. Thus, this attack allows an attacker to *upgrade* the level of trust enjoyed by the key material sent by the attacker such that no indication is given in the UI that a user should treat it with caution.

This is an implementation bug aided by the overall layout in `matrix-js-sdk`, where cryptographic functionality is spread across several submodules. In particular,  Element implements verification of message authentication not at decryption time but at display time.

We discuss this attack in the section "Trusted Impersonation and Confidentiality Breaks against Megolm" of our research paper.

## Impersonation to confidentiality break

An extension of the semi-trusted impersonation attack, combined with the aforementioned protocol confusion, allows an attacker and colluding homeserver to gain access to all inbound Megolm sessions their target has access to.

The Secure Secret Storage and Sharing (SSSS) module provides a means for a user's devices to share account-level secrets either asynchronously via encrypted backups on the homeserver, or synchronously using a secret sharing protocol.

When a user performs out-of-band verification between two of their devices, the newly verified device will use the SSSS module's secret sharing protocol to request some account level secrets from the verifying device. In particular, it will request a copy of the key used for server-side Megolm backups from the verifying device (server-side Megolm backups are encrypted backups of inbound Megolm sessions shared across a user's devices).

The same Olm/Megolm protocol confusion as described in the previous attack can be exploited by an attacker to impersonate the verifying device and reply to the request. This allows the attacker to set the Megolm backup key used by the newly verified device to a value of their choosing. The device will accept this key and proceed to backup inbound Megolm sessions to the homeserver. The attacker and colluding homeserver are then able to decrypt the backups, giving them access to the plaintext of every Megolm message the target device has access to.

This is an implementation bug.

We discuss this attack in the section "Trusted Impersonation and Confidentiality Breaks against Megolm" of our research paper.

## IND-CCA break

[AES-CTR](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation) is used for encryption in both the SSSS protocol and in symmetric Megolm Key Backups. However, the initialisation vector (IV) for AES-CTR is not included in the [message authentication code](https://en.wikipedia.org/wiki/Message_authentication_code) (MAC). A similar issue exists when attachments are shared. This can be exploited to break the [IND-CCA](https://en.wikipedia.org/wiki/Ciphertext_indistinguishability#IND-CCA) security of the underlying encryption scheme: an adversary is able to decrypt a challenge ciphertext by querying encryption and decryption oracles, without requesting decryption of the challenge ciphertext directly. However, in practice we do not know how to instantiate this attack and thus it, in contrast to those mentioned so far, is only of theoretical interest.

This is an issue in the Matrix specification.

We discuss this attack in the section "IND-CCA Attack on Backups" of our research paper.

## Summary

In summary, we found that Matrix and its flagship client Element as deployed provided neither authentication nor confidentiality against homeservers that actively attack the protocol, i.e. its end-to-end encryption fell short of the security guarantees expected from it.

# Remedies/Fixes

**The Matrix developers report on these vulnerabilities, fixes and countermeasures on their [blog](https://matrix.org/blog/category/security).**

We disclosed our attacks to the Matrix developers between 20 May 2022 and 6 July 2022. They acknowledge these as vulnerabilities except for one of our attacks on confidentiality (homeserver control of room members, see below) which they consider as an accepted risk (but aim to mitigate regardless). We coordinated a public vulnerability disclosure for the 28 September 2022, to coincide with the first set of countermeasures. These should provide immediate fixes (to varying degrees) for our attacks. At the time of public disclosure, the Matrix specification and Element will *not* be vulnerable to 

- the attack against out-of-band verification, 
- the semi-trusted impersonation attack, 
- the trusted impersonation attack and 
- the impersonation to confidentiality attack. 

A second set of countermeasures is currently in the design phase, which aim to provide complete fixes for every vulnerability we identified.

In particular, the attacks concerning homeserver control of room membership and user device lists will not be fixed at the time of disclosure. However, a new local per-room setting will be added alongside the disclosure in order to mitigate the homeserver’s control of user device lists. In the long-term, the Matrix developers plan to develop fixes for both of these attacks. 

A fix for the IND-CCA break will also be distributed at a later date. Since the IND-CCA break appears not to be practically exploitable, this should not affect users.

# Discussion

On the one hand, some of our attacks highlight a rich attack surface by “chaining” different attacks to achieve their goals. In particular, we compose a “weak” authentication break which exploits missing verifications, a stronger authentication break which exploits a protocol confusion aided by the implementation design choice to check cryptographic properties at display rather than receiving time, and a MITM attack that breaks confidentiality by convincing a target to use an adversary controlled key as backup.

On the other hand, our attacks are well distributed across the different parts of the overall cryptographic core of the Matrix protocol. In particular, we show that the Matrix specification offers no cryptographic guarantees of confidentiality by design against a malicious homeserver which may trivially add new users and devices to a room, that an identifier confusion in a separate protocol allows to break authentication and thus confidentiality even for the lowest level Olm channels, and that the key backup scheme in yet another subsystem does not achieve formal IND-CCA security.

Besides the observed implementation and specification errors, these vulnerabilities highlight a lack of a unified and formal approach to security guarantees in Matrix. Rather, the specification and its implementations seem to have grown “organically” with new sub-protocols adding new functionalities and thus inadvertently subverting the security guarantees of the core protocol. This suggests that, besides fixing the specific vulnerabilities reported here, the Matrix/Megolm specification will need to receive a [formal security analysis](https://en.wikipedia.org/wiki/Provable_security) to establish confidence in the design.

Finally, our attacks are against a setting where Matrix aims to provide the strongest guarantees, i.e. where every device and user have performed out-of-band verification. If this condition is not satisfied, even for one device or user, then “all bets are off” and e.g. impersonation becomes trivial. While Element already supports the option of refusing to send messages to unverified devices it does not reject messages from such devices. Thus, unless a client-side option is provided to reject all communication from unverified devices or rooms with such devices within them, Matrix clients will not provide a secure chat environment regardless of cryptographic guarantees provided for verified devices.

# Anticipated Questions & Answers

## Are these attacks design flaws in the Matrix specification?

We will explain this one by one by using the name of the attacks previously defined:

a. **Simple confidentiality break**: The root cause of this attack is the fact that room management messages are not authenticated, which is a design flaw in the protocol itself, as no mechanism was specified for authentication of such messages.

b. **Attack against out-of-band verification**: This attack exploits an insecure implementation choice enabled by a design flaw in the specification as there is no domain separation enforced there.

c. **Semi-trusted impersonation**: This is mostly implementation bug supported by a lack of guidance on the processing of incoming key shares in spec.

d. **Trusted impersonation**: This is an implementation error as no check is performed to check whether Olm is used for encryption or not.

e. **Impersonation to confidentiality break**:  This is an implementation error as no check is performed to check whether Olm is used for encryption or not.

f. **IND-CCA break**: This theoretical attack exploits a protocol design flaw.

## What is a Matrix's homeserver?

As specified [here](https://matrix.org/faq/#what-is-a-haomeserver%3F), a Matrix's homeserver is a server that stores the communication history and account information for a client, and shares data with the wider Matrix ecosystem by synchronising communication history with other homeservers.

The end-to-end encryption in Matrix should protect users from malicious homeservers (whether that is their own homeserver or one that is federating with their own).

## What is Element?

When relying on implementation specific behaviour, our attacks target the Matrix standard as implemented by the `matrix-react-sdk` and `matrix-js-sdk` libraries. These libraries provide the basis for the [Element](https://element.io/) flagship client, which is also sometimes referred to as the client [reference implementation](https://en.wikipedia.org/w/index.php?title=Matrix_(protocol)&oldid=1106244486) by third parties. Its website reports [42M+ users](https://web.archive.org/web/20220926114259/https://element.io/about) and usage by the French, German & US governments. While plans exist to replace Element in its current form in the future, it is currently the default client [recommended](https://web.archive.org/web/20220922112533/https://matrix.org/docs/projects/try-matrix-now/) by Matrix.

## What is Olm?

[Olm](https://matrix.org/docs/projects/other/olm) is an implementation of the [Double Ratchet protocol](https://signal.org/docs/specifications/doubleratchet/), as specified by Signal, and of a modified 3DH key exchange protocol.
Olm is used in Matrix to provide a channel by which key material is shared in a pair-wise fashion.

## What is Megolm?

[Megolm](https://gitlab.matrix.org/matrix-org/olm/blob/master/docs/megolm.md) is a protocol used for group conversations (all chats in Matrix are formally group chats). Each Megolm channel is a unidirectional channel, used to send payload information from one device to all other devices in a group. When using Megolm, senders and receivers periodically symmetrically ratchet the shared secret state forward, aiming to achieve [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy). Note, however, that the specification allows implementations to keep old copies of the ratchet on the receiving side which invalidates forward secrecy guarantees. In addition, senders can periodically generate a new (and independent) Megolm secret state, and send it to the receiving devices in the room via Olm, thus aiming to achieve some form of post-compromise security.

The Megolm protocol has not been formally analysed.

## Regarding the 'Simple confidentiality break', can't users detect when the homeserver adds a new fake user(s)/device(s) to a room?

When a *user* is added to a room, this will be displayed as an event in the timeline, and is thus detectable by users. However, such a detection requires careful manual membership list inspection from users and to participants, this event appears as a legitimate group membership event. In particular, in sufficiently big rooms such an event is likely to go unnoticed by users.

In environments where cross-signing and verification are enabled, adding a new *unverified user* adds a warning to the room to indicate that unverified devices are present. However, it is possible for a homeserver to add a verified user to rooms without changing the security properties of the room. This allows a colluding homeserver and verified user to eavesdrop on rooms not intended for them. In other words, the warning regarding unverified devices is independent to whether the device is intended to participate in the speciﬁc room. Users may, of course, simply ignore warnings.

For example, consider any room whereby multiple privileged users are entrusted with adding new members. If a new membership event shows that a privileged member has invited someone to join, this will not be questioned. All the while, this could have been a malicious homeserver acting on behalf of the privileged member. To cover their tracks against the privileged member, the malicious homeserver need only present a different privileged member as the inviter. 

In environments where cross-signing and verification are enabled, adding an *unverified device* to the user's list of devices will alert their existing sessions to start the verification process. To avoid the notification, the homeserver can present two different versions of the device list depending on the user requesting it. When a user requests their own device list, the homeserver does not include the unverified device. When a different user requests the list, the homeserver includes an unverified device that they control. The target users' devices will not be aware that a new, unverified device has been added to their account. Therefore, their clients will not present the verification dialog.

Nevertheless, adding an unverified device to the room will add a warning indicator to the room. But the same caveats as above for this countermeasure apply.

<!-- ## Aren't these attacks just like Heartbleed? -->

<!-- [Heartbleed](https://heartbleed.com/) is a serious vulnerability in the popular OpenSSL cryptographic software library. -->
<!-- It is implementation problem, i.e. programming mistake in OpenSSL. -->
<!-- The RFC 6520 Heartbeat Extension checks for TLS/DTLS secure communication links by allowing a party at one end of a connection to send a Heartbeat Request message, consisting of a payload along with the payload's length as a 16-bit integer. -->
<!-- The receiving computer then must send exactly the same payload back to the sender. -->
<!-- The affected versions of OpenSSL allocate a memory buffer for the message to be returned based on the length field in the requesting message, without regard to the actual size of that message's payload. -->
<!-- Because of this failure to do proper bounds checking, the message returned consists of the payload, possibly followed by whatever else happened to be in the allocated memory buffer. -->
<!-- This is an implementation bug as the length of the allocated buffer is incorrectly set. -->

<!-- Whilst some of our attacks are issues with Element and `matrix-js-sdk`'s implementation, some of our attacks *are* protocol design flaws (as previously highlighted). -->
<!-- For example, our 'Simple confidentiality break' attack relies on the fact that the specification provides no authentication for room membership messages. -->
<!-- The implementation of these messages is correct, but provides no authentication as it was never specified by the design. -->

## Does this mean that Matrix does not provide confidentiality and/or authentication?

Matrix and its implementations can, after today's fixes, provide confidentiality and authentication assurances against malicious homeservers, if users act as follows. Each user must enable cross-signing and perform out-of-band verification with each of their own devices, and with each user they interact with.[^2] They must then remain vigilant: any warning messages or icons must be spotted and investigated. In the Element user interface, this requires checking the room icon and each individual message they receive (in some cases, past messages can retroactively receive a warning). Note that such warnings could be expected behaviour (for example if the message was decrypted using a server-side Megolm backup or through the "Key Request protocol"). Users would need the expertise to investigate these warnings thoroughly and, if an issue is found, recover from it. If you follow these instructions without fail, Matrix can provide you with confidentiality and authentication.

This places an unnecessary burden on users of Matrix clients, limits the user base to those with an understanding of the cryptography used in Matrix and how it is used therein, and is impractical for daily use. The burden this places on users is unnecessary and the result of the design flaws we highlight in our work (this is our "Simple confidentiality break" attack). Whilst this issue will persist after today's fixes, a remediation is planned by the Matrix developers for a later date.

Some of our other attacks against Matrix's flagship client Element are based on implementation flaws and, thus, were able to break its confidentiality and authentication guarantees even when the steps above were followed (prior to today's patches). As of today, most of these issues should be fixed (see above), but we have not independently verified this. The Matrix developers report that other clients are not affected but, similarly, we have not independently verified this.

## Am I affected?

The Matrix developers discuss vulnerable and unaffected clients in their [blog posts](https://matrix.org/blog/category/security). We did not independently verify this.

For the avoidance of doubt, all our attacks require the collaboration of a malicious homeserver, i.e. passive or even active third parties cannot break confidentiality or authentication as all communication between homeservers and between clients and homeservers is protected by [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security).

## What recommendations do you give?

Our attacks together show a rich attack surface in Matrix from both a protocol and implementation perspective. While it is important to perform security audits of implementations, they sometimes fail to catch attacks that are present due to protocol flaws. In order to verify that confidentiality and authentication indeed can be provided, a formal security analysis of the protocol design is required.

# Footnotes

[^1]: Our analysis is based on, and our proof-of-concept attacks tested against, Element Web at [commit #479d4bf6](https://github.com/vector-im/element-web/tree/479d4bf64d97adea9611644695cdb373647fc644) with `matrix-react-sdk` at [commit #59b9d1e8](https://github.com/matrix-org/matrix-react-sdk/tree/59b9d1e8189b00bde440c30a962d141a1ebfa5a0) and `matrix-js-sdk` at [commit #4721aa1d](https://github.com/matrix-org/matrix-js-sdk/tree/4721aa1d241a46601601259ec7ca6db9ff1bb5fb).

[^2]: It is common among secure messengers to require out-of-band verification to protect against active mallory-in-the-middle attacks. Matrix's cross-signing feature helps minimise how often users must perform such procedures. Thus, in this case, Matrix's design *lowers the burden* for users who do use out-of-band verification.
