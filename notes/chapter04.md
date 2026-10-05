# `CHAP`ter 04: Identity and Access Management

## Abstract

This `CHAP`ter covers identity and access management (IAM), the framework of technologies and policies that identifies users, authenticates their claimed identities, and authorizes their access to system resources. The `CHAP`ter describes the five authentication factors, one-time passwords, hardware-based security modules, and biometric systems with their error metrics. The `CHAP`ter then examines the major authentication protocols: `PAP`, `CHAP`, Kerberos, `EAP` with IEEE 802.1X, RADIUS, TACACS+, and the Internet standards SAML, OpenID, and OAuth. The `CHAP`ter closes with account policies and controls and the five access control models: DAC, MAC, RBAC, ABAC, and rule-based access control.

## Objectives

- Define identity and access management and its three objectives: identification, authentication, and authorization.
- Describe the five authentication factors and distinguish single-factor, two-factor, and multi-factor authentication, plus the two types of knowledge-based authentication.
- Explain one-time passwords, the difference between TOTP and HOTP, and software-based and hardware-based OTP generation and delivery.
- Describe the Trusted Platform Module, the hardware security module, and the secure enclave, and their role in device authentication.
- Describe the seven biometric factors and the three biometric error metrics: FRR, FAR, and CER.
- Compare the `PAP` and `CHAP` authentication protocols.
- Explain Kerberos ticket-based authentication with the Key Distribution Center, Ticket-Granting Ticket, and Service Ticket.
- Describe the `EAP` framework, its four message types, the four common `EAP` methods, and IEEE 802.1X port-based access control.
- Compare the RADIUS and TACACS+ protocols for centralized authentication, authorization, and accounting.
- Distinguish the roles of SAML, OpenID, and OAuth for authentication and authorization on the Internet.
- Describe the three password policies, password storage with hashing, salting, and the four password-hashing KDFs, the five account types, account controls such as geolocation and time-of-day restrictions, and account maintenance practices.
- Compare the five access control models: DAC, MAC, RBAC, ABAC, and rule-based access control.
- Explain filesystem permissions in Linux/Unix and Windows, privileged access management, and conditional access.

## Content

### 1. Identity and Access Management Principles

Identity and access management (IAM), also called identity management (IdM), is a framework of technologies and policies for managing user identities in a system and controlling user access to the system's resources. IAM pursues three objectives: the identification, authentication, and authorization of users.

#### **Identification**

A user is typically a person but can also be an application, a process, or a device. Identification is the act of a user claiming an identity. A user can claim an identity through several means:

- **Username:** a unique sequence of characters assigned to a user.
- **Certificate:** a digital credential that binds a user's identity to a cryptographic key.
- **Token:** a physical device assigned to a user that generates a unique code.
- **SSH key:** a cryptographic key pair owned by a user.
- **Smart card:** a card with an embedded microchip assigned to a user.

Identity proofing is the process of verifying a user's identity during account creation. For example, registering for an online banking account requires verification against a government-issued ID. A system associates an identity with attributes, where an attribute is a specific characteristic of the identity such as a name, an email address, a location, or a password.

#### **Authentication**

Authentication is the act of verifying a user's claim to an identity. A user proves a claim by presenting evidence such as possession of a device, presence at a location, or knowledge of a password or PIN. These evidence types are called authentication factors. The user and the system exchange authentication information through an authentication protocol, a communications protocol designed for securely transferring authentication information between two parties. Authentication protocols covered in this `CHAP`ter include `PAP`, `CHAP`, Kerberos, EAP, IEEE 802.1X, RADIUS, and TACACS+.

#### **Authorization**

Authorization is the act of granting a user access to a system resource, and occurs after the system identifies and authenticates the user. Access control involves a subject and an object. A subject is an entity that wants access to a resource, and an object is the resource the subject wants to access. An access control model, also called an authorization model, is a set of technology-independent rules for controlling access to an object by a subject. The five commonly used access control models are discretionary access control (DAC), mandatory access control (MAC), role-based access control (RBAC), attribute-based access control (ABAC), and rule-based access control.

### 2. Authentication Factors

An authentication factor is a type of evidence a user presents to prove a claimed identity. Five common authentication factors exist:

- **Knowledge factor:** a piece of information the user knows. Something you know. Common examples are a password, a personal identification number (PIN), a passphrase, and an answer to a security question. For example, entering a 4-digit PIN at an ATM proves knowledge of the PIN. The knowledge factor is the oldest and most widely used factor, and also the weakest when used alone: a password can be guessed, shared, stolen in a data breach, or recovered by a brute-force attack. For example, the 2019 Collection #1 breach disclosed more than 770 million email addresses and 21 million passwords, which attackers could replay against other systems.
- **Possession factor:** something the user owns. Something you have. Common examples are a smart card, a mobile phone, a hardware security key, and an OTP generator device. For example, a bank's ATM card is a possession factor: the card is useless without the matching PIN (a knowledge factor), which is why ATM transactions combine the two factors. A hardware security key such as a YubiKey proves possession by performing a cryptographic operation with a private key stored on the key, so the credential never leaves the device. A possession factor resists remote guessing attacks but can be lost or stolen, so systems pair the factor with a PIN or a biometric.
- **Inherence factor:** a unique physical characteristic of the user. Something you are. Common examples are a fingerprint, a voiceprint, a retina or iris pattern, and a face. For example, signing in to a smartphone with a fingerprint through Apple Touch ID or a face through Apple Face ID uses an inherence factor. An inherence factor cannot be forgotten, shared, or transferred, but the factor cannot be changed if compromised: a user can reset a stolen password but cannot reset a stolen fingerprint.
- **Location factor:** information about the user's current location, determined from GPS coordinates or the IP address of the user's device. Somewhere you are. For example, a corporate VPN can accept logins only from IP addresses inside the company's home country, and a banking application can flag a login from a foreign country minutes after a domestic login as impossible travel. The location factor rarely stands alone and instead strengthens or weakens a risk decision made from other factors.
- **Behavior factor:** an action the user performs. Something you can do. Common examples are a hand gesture, a specific pattern drawn on a device screen, and keystroke dynamics such as typing rhythm and speed. For example, Android's pattern lock requires the user to draw a predefined shape across a grid of dots, and a behavioral biometric system can verify a user continuously by analyzing how the user types a password rather than what the password is.

Two factor types strengthen each other only when the factors come from different categories. Two passwords remain single-factor authentication because both are knowledge factors, while a password plus a hardware security key counts as two factors because knowledge and possession are different categories.

#### **Single-Factor and Multi-Factor Authentication**

Single-factor authentication (SFA) verifies a claimed identity with one authentication factor. SFA is the most basic form of authentication and is most often implemented as a password. For example, a secured system using SFA may require only a smart card (possession factor) or only a password (knowledge factor).

Two-factor authentication (2FA) verifies a claimed identity with two different authentication factors. For example, a secured system using 2FA may require a password (knowledge factor) and a verification code sent to the user's mobile phone (possession factor). Smart card authentication is a form of 2FA: the user possesses the smart card (possession factor) and knows the card's PIN (knowledge factor).

Multi-factor authentication (MFA) verifies a claimed identity with two or more authentication factors, so 2FA is a type of MFA. Adding authentication factors increases security but degrades user experience and reduces system usability.

#### **Knowledge-Based Authentication**

Knowledge-based authentication (KBA) validates a claimed identity by testing knowledge of the user's personal information. KBA commonly appears in multi-factor authentication systems. Two types of KBA exist:

- **Static KBA:** a pre-agreed set of shared questions between the authentication system and the user, also called shared secrets or shared secret questions. The user selects answers to a set of static questions during registration, such as the name of the user's best friend or the user's birth city.
- **Dynamic KBA:** knowledge questions that the user does not set beforehand. The authentication system compiles questions from the user's public and private information, such as credit reports and financial transaction history. Examples include the user's personal loan balances or credit card limits.

Dynamic KBA is more effective than static KBA because the questions reference both current and historical user information across a wider range. A dynamic KBA system may also generate diversionary questions designed to trick an impostor, such as presenting a list of phone numbers and asking the user to select a number the user never had.

### 3. One-Time Passwords

A one-time password (OTP) is an authentication code that can be used only once. An OTP is a 6- to 10-digit code commonly deployed in 2FA systems. An OTP is more secure than a static password: a user cannot share an OTP for later reuse, cannot reuse an OTP across systems, and does not need to memorize a complex password. Two types of OTP exist:

- **Time-based one-time password (TOTP):** an OTP that changes periodically, generated from a timer and a secret key. A change occurs at each timestep, an increment of time between 30 and 180 seconds, and a TOTP remains valid for the duration of one timestep.
- **HMAC-based one-time password (HOTP):** an OTP that changes based on an event, also called event-based OTP. HOTP is generated from a counter and a secret key, and the counter increments on an event such as a button press on the generator device. An HOTP remains valid until used for authentication or until a new HOTP is generated.

#### **Software-Based OTP**

An authentication application, also called a software token, is an application that generates OTPs. For example, Google Authenticator is a mobile authentication application that generates TOTP and HOTP codes for Google's 2-Step Verification. Before use, the authentication application must be configured with an authentication server: the server sends a secret key to the application, and the application uses that secret key to generate OTPs for all future authentication requests to that server.

An OTP can also be delivered by email, SMS, or phone call. A push notification is a notification an authentication server sends to a mobile device associated with a user. The push notification informs the user of an authentication attempt with the user's credentials, and the user approves the attempt by performing a specific action, such as pressing a button on the phone or opening an application. A push notification validates identity through a possession factor: the user's mobile phone.

#### **Hardware-Based OTP**

A security token, also called a token key, security key, or password key, is a hardware device that generates OTPs. A security token holds an encoded secret key shared only with the authentication server. Because the token never stores an OTP on a networked device such as a mobile phone or an email client, the token is more secure than OTP delivery over SMS or email. For example, RSA Security's SecurID is a security token.

An OTP can also be pre-generated for situations where no hardware or software token and no network connection exist. A static code is a pre-generated OTP that a hardware or software OTP generator produces and that the user saves on secured storage media for later use.

### 4. Hardware Security: TPM, HSM, and Secure Enclave

Hardware devices themselves can be authenticated so that unauthorized operations are not performed on data and systems. Two hardware-based technologies support device authentication: the Trusted Platform Module and the hardware security module.

**A Trusted Platform Module (TPM)** is a secure processor that performs cryptographic operations. A TPM can be embedded on a computer motherboard or installed in a motherboard's TPM port. Each TPM carries a burned-in RSA key pair that can authenticate the hardware device. A TPM can store cryptographic keys, passwords, and certificates. A TPM is tamper-resistant and helps prevent unauthorized modifications to firmware and software as part of a secure (trusted) boot process.

An operating system can use a TPM for full disk encryption (FDE), the encryption of an entire drive including all user files, operating system files, and software programs. For example, Microsoft Windows BitLocker is a full disk encryption product that can store a drive's encryption key in a TPM.

**A hardware security module (HSM)** is a tamper-resistant external device or plug-in expansion card that provides cryptographic services. An HSM can store a hardware device's certificate, cryptographic keys, and passwords. An HSM contains one or more cryptoprocessors, dedicated microprocessors that perform cryptographic operations such as encryption, and uses the cryptoprocessors to create, secure, store, and manage encryption keys. For example, the payment card industry uses HSMs for real-time authentication and authorization of credit and debit card transactions.

A TPM and an HSM can be certified against a security standard. The Federal Information Processing Standards (FIPS) 140 publication series, known as `FIPS-140`, is the U.S. government computer security standard that specifies requirements for cryptographic modules.

#### **Secure Enclave**

A secure enclave is a tamper-resistant hardware component isolated within a system or device that provides a secure environment for cryptographic operations. A secure enclave keeps sensitive data encrypted and inaccessible to other software, including the operating system. Secure enclaves help devices comply with data protection regulations such as regulations for digital rights management and secure payment processing.

### 5. Biometric Authentication

Biometrics are measurements of the unique characteristics of an individual, such as a fingerprint or a voice. Two types of biometrics exist:

- **Physical biometrics:** measurements of physical characteristics, such as a fingerprint, a palmprint, a face, a retina, an iris, and vein patterns.
- **Behavioral biometrics:** measurements of behavioral characteristics, such as voice, gait, signature, and keystroke dynamics.

Biometric authentication uses physical and behavioral biometrics to verify a claimed identity. Biometric authentication is more secure than knowledge-based or token-based authentication: a password or PIN can be shared or guessed, and a token can be lost or stolen, but a biometric is unique to an individual and cannot be transferred, lost, or forgotten. A biometric is nearly impossible to duplicate.

#### **Biometric Factors**

A biometric factor, also called an inherence factor, is a unique physical or behavioral characteristic of an individual. Seven biometric factors are commonly used for authentication:

- **Fingerprint:** the pattern of ridges and valleys on an individual's finger, scanned with an ultrasonic, optical, or capacitive scanner.
- **Retina pattern:** the blood vessel pattern in the retina, the layer of nerve cells lining the back of the eye, scanned with low-energy infrared light.
- **Iris pattern:** the pattern of the iris, the colored portion of the eye surrounding the pupil, scanned with video camera technology.
- **Facial recognition:** measurement of distances between points on an individual's face, such as jawline length, cheekbone shape, and nose width, to create a faceprint, a digitally recorded representation of the face. A video, infrared, or thermal camera performs the scan.
- **Voice recognition:** measurement and analysis of the rhythms, patterns, and sounds of an individual's voice through a microphone.
- **Vein recognition:** capture and analysis of blood vessel patterns with an infrared-light scanner. Unlike a fingerprint scanner, a vein scanner does not touch the skin.
- **Gait analysis:** the study of an individual's walking pattern, including body movements and muscle activity during motion.

#### **Biometric Errors**

The performance of a biometric system is measured with three metrics:

- **False rejection rate (FRR):** the percentage of valid biometric measures the system rejects, also called a type I error.
- **False acceptance rate (FAR):** the percentage of invalid biometric measures the system accepts, also called a type II error.
- **Crossover error rate (CER):** the rate at which FRR and FAR are equal, also called the equal error rate (EER).

The effectiveness or efficacy rate of a biometric system measures the system's ability to minimize false acceptance and prevent false rejection. The FIDO Alliance, which certifies biometric systems, has established an FRR threshold of 3% (3 in 100) and a FAR threshold of 0.01% (1 in 10,000).

Lowering the false rejection rate raises the false acceptance rate. The relative operating characteristic (ROC) graph visualizes this trade-off between FRR and FAR, and the CER is the point on the graph where the FRR and FAR curves intersect. CER serves to compare the accuracy of different biometric systems: the system with the lowest CER is the most accurate.

### 6. Authentication Protocols: `PAP` and `CHAP`

#### **Password Authentication Protocol (`PAP`)**

Password authentication protocol (`PAP`) authenticates a client to a server over a point-to-point connection. `PAP` authenticates only once per session, at the time of initial connection establishment. `PAP` uses a two-way handshake: the client sends a username and password to the server, and the server replies with an authentication acknowledgement (ack) if the credentials are correct or a negative acknowledgement (nak) if the credentials are incorrect. `PAP` does not encrypt usernames and passwords, so `PAP` is not considered a secure authentication protocol.

```text
PAP two-way handshake (performed once, at connection establishment):

Client                                   Server
  |                                        |
  |---- username + password (cleartext) -->|
  |                                        | verify credentials
  |                                        |
  |<---------------- ack ------------------|  credentials correct
  |<---------------- nak ------------------|  credentials incorrect
  |                                        |
 (no re-authentication for the rest of the session)
```

#### **Challenge Handshake Authentication Protocol (`CHAP`)**

Challenge handshake authentication protocol (`CHAP`) uses a shared secret to authenticate a client to a server over a point-to-point connection. Unlike `PAP`, `CHAP` periodically re-authenticates the client during the communication session.

`CHAP` uses a three-way handshake. The server sends a randomly generated challenge string to the client. The client combines the challenge string with the secret shared with the server, computes the hash value of the combined string, and sends the hash value to the server. The server compares the received hash value with the server's own calculated hash value of the challenge string and shared secret. Equal hash values authenticate the client. Unequal hash values fail the authentication attempt.

```text
CHAP three-way handshake (repeated periodically during the session):

Client                                   Server
  |                                        |
  |<--------- random challenge string -----|
  |                                        |
  |  compute H(challenge + shared secret)  |  compute H(challenge + shared secret)
  |                                        |
  |--------- hash value (response) ------->|
  |                                        | compare the two hash values
  |                                        |
  |<---------------- success --------------|  hash values equal
  |<---------------- failure --------------|  hash values unequal
  |                                        |
 (the shared secret itself never crosses the network)
```

The password never travels over the connection in `CHAP`. An attacker capturing the exchange sees only the random challenge and the hash value, and the random challenge prevents replaying a captured hash in a later session.

### 7. Authentication Protocols: Kerberos

Kerberos is an authentication protocol that uses a ticket-based mechanism to authenticate a user and grant the user access to a network service. Kerberos uses UDP port 88 by default. For example, Kerberos is the default authentication protocol in Windows Server 2019. A Kerberos authentication involves three entities:

- **Server:** a host running the service the user wants to access.
- **Client:** the machine from which the user requests access.
- **Key Distribution Center (KDC):** a trusted third party that authenticates the user and grants access to services. A KDC consists of an Authentication Server (AS), which authenticates users, and a Ticket-Granting Server (TGS), which issues tickets for service access.

#### **Kerberos Authentication**

A user who wants to authenticate to the network sends an authentication request message containing identity information and credentials to the Authentication Server. The AS validates the user's identity and returns a Ticket-Granting Ticket (TGT), a ticket that lets the user request service access from the TGS.

When the user wants to access a service, the user sends the TGT and the service's identification number to the TGS. The TGT proves to the TGS that the user was previously authenticated. The TGS responds with a Service Ticket (ST), an encrypted message that proves the user is authorized to access the service. An ST names one service and carries a limited validity period, and the client caches the ST and reuses the ticket until expiration. A user who wants to access a different service must request a new ST from the TGS.

The full exchange runs in three stages. In the first stage the user proves identity to the AS and receives a TGT plus a session key for talking to the TGS. In the second stage the user presents the TGT to the TGS and receives an ST plus a session key for talking to the service. In the third stage the user presents the ST to the server and gains access. A session key is a temporary symmetric key shared by two parties for one exchange. Timestamps inside each message prevent replay attacks: a ticket or authenticator with an old timestamp is rejected.

```text
Kerberos ticket exchange (all traffic on UDP port 88):

Stage 1: Authentication Service exchange (user to AS)

Client                                        AS (in the KDC)
  |                                              |
  |--- (1) AS-REQ: user ID, service ID, -------->|
  |      timestamp (pre-authentication data      |
  |      encrypted with the user's key derived   |
  |      from the user's password)               |
  |                                              | verify the timestamp:
  |                                              | only the real user can
  |                                              | encrypt with that key
  |                                              |
  |<-- (2) AS-REP: ------------------------------|
  |      (a) TGT: user ID, validity period, and  |
  |          a client/TGS session key, encrypted |
  |          with the TGS secret key (the client |
  |          cannot read the TGT)                |
  |      (b) client/TGS session key, encrypted   |
  |          with the user's key                 |
  |                                              |
  |  client derives its key from the password    |
  |  and decrypts (b)                            |

Stage 2: Ticket-Granting Service exchange (user to TGS)

Client                                        TGS (in the KDC)
  |                                              |
  |--- (3) TGS-REQ: ---------------------------->|
  |      (a) the TGT from step 2                 |
  |      (b) authenticator: user ID and          |
  |          timestamp, encrypted with the       |
  |          client/TGS session key              |
  |      (c) requested service ID                |
  |                                              | decrypt the TGT with the
  |                                              | TGS secret key, recover
  |                                              | the session key, decrypt
  |                                              | the authenticator, and
  |                                              | check the timestamp
  |                                              |
  |<-- (4) TGS-REP: -----------------------------|
  |      (a) ST: user ID, client address,        |
  |          validity period, and a client/      |
  |          server session key, encrypted with  |
  |          the service's secret key (the       |
  |          client cannot read the ST)          |
  |      (b) client/server session key,          |
  |          encrypted with the client/TGS       |
  |          session key                         |
  |                                              |
  |  client decrypts (b)                         |

Stage 3: Client/Server exchange (user to service)

Client                                        Server
  |                                              |
  |--- (5) AP-REQ: ----------------------------->|
  |      (a) the ST from step 4                  |
  |      (b) authenticator: user ID and          |
  |          timestamp, encrypted with the       |
  |          client/server session key           |
  |                                              | decrypt the ST with the
  |                                              | service's secret key,
  |                                              | recover the session key,
  |                                              | decrypt the authenticator,
  |                                              | and check the timestamp
  |                                              |
  |<-- (6) AP-REP (optional, for mutual --------|
  |      authentication): timestamp from the     |
  |      authenticator, encrypted with the       |
  |      client/server session key               |
  |                                              |
  |======= secure session with the service ======|
```

The design keeps secrets off the network. The user's password never leaves the client: the password only derives the key that decrypts the AS-REP. The TGS and the service each read their tickets with their own secret keys, so the client cannot forge or alter a ticket. Mutual authentication in step 6 also proves the server's identity to the client, protecting the client against a rogue server.

### 8. Authentication Protocols: `EAP` and `IEEE 802.1X`

#### **Extensible Authentication Protocol (`EAP`)**

The Extensible Authentication Protocol (`EAP`) is an authentication framework for transporting different types of authentication protocols. `EAP` supports multiple authentication methods, including certificate-based, password-based, and multi-factor authentication. `EAP` defines the format of authentication messages, and four `EAP` message types exist:

- **`EAP` Request:** a message from a server to a client requesting information.
- **`EAP` Response:** a message from a client to a server replying to an `EAP` Request.
- **`EAP` Success:** a message from a server to a client when authentication succeeds.
- **`EAP` Failure:** a message from a server to a client when authentication fails.

An authentication method supported by `EAP` implements these messages in the method's own messaging format, so a new authentication method becomes compatible with existing wireless or point-to-point connection technologies. Over 40 `EAP` authentication methods have been defined. Four commonly used `EAP` methods exist:

- **`EAP-TLS`:** Extensible Authentication Protocol-Transport Layer Security uses TLS and certificates for mutual authentication. EAP-TLS requires both a server certificate and a client certificate.
- **`EAP-FAST`:** Extensible Authentication Protocol-Flexible Authentication via Secure Tunneling uses a Protected Access Credential (PAC) to establish a TLS tunnel between server and client. A PAC is a security credential generated by the server that holds client-specific information. EAP-FAST supports but does not require certificates.
- **`EAP-TTLS`:** Extensible Authentication Protocol-Tunneled Transport Layer Security uses TLS and a server certificate to establish a secure tunnel, then authenticates the client through the tunnel with any authentication protocol, including legacy password methods such as `PAP`. EAP-TTLS requires a server certificate but not a client certificate.
- **`PEAP`:** Protected Extensible Authentication Protocol encapsulates `EAP` messages within an encrypted and authenticated TLS tunnel. `PEAP` requires a server certificate but not a client certificate.

#### **Comparing `EAP-TLS`, `EAP-FAST`, `EAP-TTLS`, and `PEAP`**

The four methods differ in two design decisions: what establishes the TLS tunnel, and what authenticates the client. `EAP-TLS` uses certificates on both sides and builds no separate tunnel, because the certificate exchange itself is the authentication. The other three methods first build a TLS tunnel and then run the client authentication inside the tunnel, so the client's credentials never cross the network in cleartext. The methods differ in what builds the tunnel: a server certificate in `EAP-TTLS` and `PEAP`, and a PAC (or an optional certificate) in `EAP-FAST`.

| Method | Server certificate | Client certificate | Client credential | Typical deployment |
| --- | --- | --- | --- | --- |
| `EAP-TLS` | Required | Required | Certificate only, no password | Enterprises that issue a certificate to every managed device |
| `EAP-TTLS` | Required | Not required | Any inner method, including legacy `PAP` passwords | University networks such as eduroam |
| `PEAP` | Required | Not required | An inner `EAP` method, usually `EAP-MSCHAPv2` | Corporate Wi-Fi where users log in with a username and password |
| `EAP-FAST` | Optional | Not required | Credentials inside a PAC-established tunnel | Cisco wireless deployments |

`EAP-TLS` diagram: both sides present certificates, and no tunnel or inner authentication exists.

```text
EAP-TLS (mutual certificate authentication, no tunnel):

Supplicant                                   Server
  |                                            |
  |<--------------- server certificate --------|
  |  verify server certificate                 |
  |--------------- client certificate -------->|
  |                                            |  verify client certificate
  |<--------------- EAP Success ---------------|
  |                                            |
 (no tunnel: the certificate exchange is the authentication)
```

`EAP-TTLS` diagram: the server certificate builds the tunnel, and any client authentication method runs inside, including legacy password protocols such as `PAP`.

```text
EAP-TTLS (server certificate builds the tunnel, any inner method):

Supplicant                                   Server
  |                                            |
  |<--------------- server certificate --------|
  |  verify server certificate                 |
  |========== TLS tunnel established ==========|
  |------- inner authentication -------------->|
  |        (PAP, CHAP, or MSCHAPv2 password    |
  |         exchange, protected by the tunnel) |
  |<--------------- EAP Success ---------------|
```

`PEAP` diagram: the server certificate builds the tunnel, and a second, inner `EAP` method runs inside. `EAP-TTLS` can carry any authentication protocol inside the tunnel, while `PEAP` carries only another `EAP` method.

```text
PEAP (server certificate builds the tunnel, inner EAP method inside):

Supplicant                                   Server
  |                                            |
  |<--------------- server certificate --------|
  |  verify server certificate                 |
  |========== TLS tunnel established ==========|
  |------- inner EAP method ------------------>|
  |        (EAP-MSCHAPv2 or EAP-TLS,           |
  |         protected by the tunnel)           |
  |<--------------- EAP Success ---------------|
```

`EAP-FAST` diagram: a PAC builds the tunnel instead of a certificate, so the method deploys without a public key infrastructure.

```text
EAP-FAST (PAC builds the tunnel, certificates optional):

Supplicant                                   Server
  |                                            |
  |<--------------- PAC (generated by the -----|
  |        server, client-specific)            |
  |========== TLS tunnel established ==========|
  |------- client credentials inside --------->|
  |        the tunnel                          |
  |<--------------- EAP Success ---------------|
```

Real deployments follow the certificate cost trade-off. `EAP-TLS` gives the strongest authentication because no password exists to guess or phish, but the enterprise must issue, distribute, and renew a certificate on every client device. The eduroam university network commonly deploys `EAP-TTLS` with `PAP` or `PEAP` with `MSCHAPv2`, so thousands of students can log in with a username and password while the password stays protected inside the tunnel. Cisco developed `EAP-FAST` as a replacement for the vulnerable LEAP protocol, and `EAP-FAST` remains common in Cisco wireless deployments because the PAC removes the need for client certificates.

#### **IEEE 802.1X**

`IEEE 802.1X` is an IEEE standard for port-based access control, used for passing `EAP` messages over a wired or wireless network. `EAP` over LAN (`EAPOL`) is the encapsulation of `EAP` over `IEEE 802.1X`. `IEEE 802.1X` allows a user to connect to a network port only after the user authenticates to the network. `IEEE 802.1X` authentication involves three entities:

- **Supplicant:** the user or device that wants to authenticate to the network.
- **Authentication server:** the server that authenticates the supplicant and makes the access control decision.
- **Authenticator:** a device that acts as a proxy for the supplicant and controls the supplicant's communication with the authentication server.

`IEEE 802.1X` uses `EAP` to carry communication among the supplicant, the authenticator, and the authentication server. In a typical `802.1X` deployment the authentication server is a `RADIUS` server: the authenticator relays the supplicant's `EAP` messages to the `RADIUS` server, which makes the access control decision covered in the next section.

### 9. Authentication Protocols: RADIUS and TACACS+

#### **RADIUS**

Remote Authentication Dial-In User Service (RADIUS) is a networking protocol for centralized authentication, authorization, and accounting (AAA) services. RADIUS combines authentication and authorization into a single service. RADIUS uses **UDP port 1812** for authentication and authorization and **UDP port 1813** for accounting. For example, Windows Server 2019 can be configured to use RADIUS for authenticating and authorizing users.

RADIUS exchanges messages between a RADIUS server and a RADIUS client. The RADIUS server provides authentication and authorization services. The RADIUS client is a network access server (NAS) that acts as an intermediary between a user requesting a connection and the RADIUS server. A NAS, also called a remote access server (RAS), is an access gateway between an untrusted external network such as the Internet and a trusted internal network.

#### **RADIUS Authentication**

A user who wants to authenticate to the network connects to a RADIUS client. The RADIUS client prompts the user for credentials and sends an Access Request message, an authentication request containing the user's credentials, to the RADIUS server. The RADIUS server validates the credentials using an authentication protocol such as `CHAP` and responds with one of three messages:

- **Access Reject:** sent when the user's credentials are not valid.
- **Access Challenge:** sent when the server requires additional information from the user, such as a PIN or a second password.
- **Access Accept:** sent when the user's credentials are valid.

After successful authentication, the RADIUS server grants the user access to the user's authorized network resources.

```text
RADIUS authentication (UDP 1812, accounting runs separately on UDP 1813):

User                      RADIUS client (NAS)              RADIUS server
  |                             |                                |
  |--- connect ---------------->|                                |
  |                             |                                |
  |<-- prompt for credentials --|                                |
  |--- username + password ---->|                                |
  |                             |--- (1) Access Request -------->|
  |                             |      (user credentials, only   |
  |                             |       the password field is    |
  |                             |       encrypted)               |
  |                             |                                | validate credentials,
  |                             |                                | for example with CHAP
  |                             |                                |
  |                             |<-- (2) one of three responses -|
  |                             |      Access Reject: invalid    |
  |                             |        credentials             |
  |                             |      Access Challenge: more    |
  |                             |        information needed      |
  |                             |        (PIN, second password)  |
  |                             |      Access Accept: valid      |
  |                             |        credentials, plus the   |
  |                             |        user's authorization    |
  |                             |        attributes              |
  |                             |                                |
  |<-- access granted to -------|  (on Access Accept)            |
  |    authorized network       |                                |
  |    resources                |                                |
  |                             |--- (3) Accounting Request ---->|
  |        session activity     |      (session start, later     |
  |                             |      session stop)             |
```

The diagram shows the two RADIUS roles:

- *Authentication* and *authorization* travel together on UDP port 1812 in steps 1 and 2.
- *Accounting* travels separately on UDP port 1813 in step 3.
- The Access Accept message carries the user's authorization attributes, so the NAS learns what the user may access in the same exchange that authenticates the user.

#### **TACACS+**

Terminal Access Controller Access-Control System Plus (TACACS+) is a proprietary networking protocol developed by Cisco for centralized authentication, authorization, and accounting services. TACACS+ separates authentication, authorization, and accounting into distinct services, and can authenticate both users and devices. TACACS+ uses TCP port 49. Unlike RADIUS, which encrypts only the user's password, TACACS+ encrypts the entire body of each AAA packet and leaves only the header in cleartext.

A TACACS+ client is a network access node such as a NAS, and a TACACS+ server holds authentication information for users and devices. The NAS obtains the user's or device's credentials and sends an authentication request to the TACACS+ server, which responds with one of three messages:

- **Accept:** sent when the credentials are valid.
- **Reject:** sent when the credentials are not valid.
- **Error:** sent when the server detects a network error during the authentication process.

```text
TACACS+ authentication (TCP 49, entire packet encrypted):

Admin                     TACACS+ client (NAS)             TACACS+ server
  |                             |                                |
  |--- connect to device ------>|                                |
  |                             |                                |
  |<-- prompt for credentials --|                                |
  |--- username + password ---->|                                |
  |                             |--- (1) authentication request ->|
  |                             |      (user credentials, entire |
  |                             |       packet encrypted)        |
  |                             |                                | validate credentials
  |                             |                                |
  |                             |<-- (2) one of three responses -|
  |                             |      Accept: valid credentials |
  |                             |      Reject: invalid           |
  |                             |        credentials             |
  |                             |      Error: network error      |
  |                             |        during authentication   |
  |                             |                                |
  |<-- login granted -----------|  (on Accept)                   |
  |                             |                                |
  |--- command (for example, -->|                                |
  |    "configure interface")   |                                |
  |                             |--- (3) authorization request ->|
  |                             |      (is this user allowed to  |
  |                             |       run this command?)       |
  |                             |<-- (4) authorization response -|
  |                             |      (permit or deny)          |
  |                             |                                |
  |                             |--- (5) accounting record ----->|
  |        each command is      |      (who ran what, and when)  |
  |        logged               |                                |
```

The diagram shows the separation that distinguishes TACACS+ from RADIUS. Authentication happens in steps 1 and 2. Authorization is a separate exchange in steps 3 and 4, and because TACACS+ can authorize each command individually, an administrator may run monitoring commands while configuration commands stay denied. Accounting in step 5 records each command, which produces a per-command audit trail. This per-command control explains why TACACS+ is the common choice for device administration, while RADIUS, which combines authentication and authorization into one exchange, fits network access for end users.

TACACS+ is commonly used to authenticate administrator accounts to network devices such as switches and routers.

### 10. Authentication and Authorization on the Internet: SAML, OpenID, and OAuth

#### **Security Assertions Markup Language (SAML)**

Security Assertions Markup Language (`SAML`) is an XML-based standard for exchanging authentication and authorization information. SAML provides single sign-on (SSO) for web-based applications and is commonly used by web portals. SSO is an authentication method that lets a user log in to multiple systems with a single identity without authenticating to each system separately. SAML defines three roles:

- **Principal:** a human user.
- **Identity provider (IdP):** an entity that creates, manages, and maintains identity information for a principal.
- **Service provider (SP):** an entity that provides a service to a principal.

SAML operates between an IdP and an SP. When a principal requests a service from an SP, the SP asks the IdP for the principal's authentication and authorization information. The IdP answers with a SAML assertion, a digitally signed XML document containing statements about the principal. Three types of SAML assertion statements exist:

- **Authentication statement:** asserts that the principal authenticated at a specific time using a specific method.
- **Attribute statement:** asserts that the principal is associated with a specific attribute, expressed as a name-value pair.
- **Authorization statement:** asserts that the principal is permitted to perform a specific action on a specific resource.

The SP makes the access control decision, whether to perform the principal's requested service, based on the IdP's assertion statements.

#### **`SAML` Example: Employee Access to Salesforce**

A concrete `SAML` single sign-on flow involves three named components. The principal is Alice, an employee using a web browser. The service provider is Salesforce, which hosts the company's customer records. The identity provider is the company's Okta portal, which holds Alice's account, password, and multi-factor settings. Alice never creates a password at Salesforce: Salesforce trusts Okta to authenticate Alice.

```text
SAML single sign-on (SP-initiated flow):

Alice's browser                Salesforce (SP)               Okta (IdP)
  |                                |                             |
  |--- (1) GET salesforce.com ---->|                             |
  |    (no session yet)            |                             |
  |                                |  build a SAMLRequest:       |
  |                                |  "authenticate this user"   |
  |<-- (2) HTTP redirect to Okta --|                             |
  |    carrying the SAMLRequest    |                             |
  |                                |                             |
  |--- (3) GET okta.com -----------+---------------------------->|
  |    with the SAMLRequest        |                             |
  |                                |                             | (4) authenticate Alice:
  |<------------------------------+---------- login page --------|     password + MFA push
  |--- credentials + MFA approval +----------------------------->|
  |                                |                             |
  |                                |                             | (5) build a digitally
  |                                |                             |     signed SAML assertion
  |                                |                             |
  |<------------------------------+---- (6) assertion returned --|
  |    in an HTML form             |                             |
  |                                |                             |
  |--- (7) browser POSTs the ----->|                             |
  |    assertion to the Salesforce |  (8) verify the signature    |
  |    Assertion Consumer Service  |      with Okta's public key, |
  |    (ACS) URL                   |      read the statements,    |
  |                                |      apply access policy     |
  |<-- (9) logged in, Salesforce --|                             |
  |    dashboard loads             |                             |
```

The browser carries every message between the SP and the IdP through redirects in steps 2 and 3 and a form POST in step 7, so Salesforce and Okta never talk to each other directly. The digital signature on the assertion in step 5 is the core trust mechanism: Salesforce holds Okta's public key from a one-time federation setup, and a forged assertion fails the signature check in step 8.

A simplified version of the assertion that Okta returns for Alice:

```xml
<saml:Assertion ID="_abc123" IssueInstant="2026-10-05T14:30:00Z">
  <saml:Issuer>https://company.okta.com</saml:Issuer>
  <ds:Signature>...</ds:Signature>          <!-- IdP's digital signature -->

  <saml:Subject>
    <saml:NameID>alice@company.com</saml:NameID>
  </saml:Subject>

  <!-- Authentication statement: who, when, how -->
  <saml:AuthnStatement AuthnInstant="2026-10-05T14:30:00Z">
    <saml:AuthnContext>password + MFA</saml:AuthnContext>
  </saml:AuthnStatement>

  <!-- Attribute statement: name-value pairs about Alice -->
  <saml:AttributeStatement>
    <saml:Attribute Name="department">
      <saml:AttributeValue>sales</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="role">
      <saml:AttributeValue>account-manager</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

Salesforce maps the assertion to the three statement types from this section. The authentication statement tells Salesforce that Alice authenticated at 2:30 PM with a password and MFA. The attribute statement carries the `name-value` pairs `department=sales` and `role=account-manager`, which Salesforce uses to place Alice in the correct profile. The authorization decision happens at Salesforce in step 8: Salesforce compares the attributes against its access policy and grants Alice access to the sales records that the account-manager role permits.

#### **OpenID**

OpenID is an open-standard protocol for decentralized authentication. OpenID lets a user log in to multiple websites without holding separate login credentials for each website, and lets a client (a website or an application) verify a user's identity without managing the user's credentials. OpenID does not rely on a central authority for authentication.

An OpenID Identity Provider (OpenID IdP) is a website that manages user identity information. For example, Google operated as an OpenID 2.0 provider until deprecating that protocol in 2015, and Google now provides federated sign-in through OpenID Connect, the successor identity layer built on OAuth. An OpenID acceptor, also called a relying party (RP), is a website that accepts OpenID authentication. A user creates an account with an OpenID IdP and uses that account to sign in to an OpenID acceptor. The acceptor redirects the user's authentication request to the IdP that manages the user's identity, and the IdP confirms the user's identity to the acceptor. OpenID does not mandate a specific authentication method: an IdP can authenticate users with legacy passwords, smart cards, or biometrics.

#### **OAuth**

OAuth is an open-standard authorization protocol. OAuth lets a user grant a client (a website or an application) access to the user's information held at other websites without sharing the user's credentials with the client. OAuth is designed for use with HTTP.

OAuth gives the client secure delegated access to the user's information on the user's behalf. An authorization server issues access tokens to the client with the approval of the user who owns the resource, and the client uses the access token to access the user's resource hosted by a resource server. For example, Google and Microsoft use OAuth to let a user share account information with a third-party application or website.

### 11. Accounts: Types and Policies

#### **Account Policies**

An account policy defines how a computer account should be created, used, maintained, disabled, and removed. A password protects an account, and a password policy is a set of rules that define a password's properties. Three common password policies exist:

- **Password complexity:** defines requirements for a password's length and character types, preventing brute-force attacks against the password.
- **Password history:** defines how often an old password can be reused, preventing password reuse, the act of reusing a previously used password.
- **Password age:** defines how often a password must be changed, also called password life-span policy. A password age policy limits the time a compromised password can be used to access an account.

#### **Password Storage: Hashing and Salting**

A system must assume the password database will be stolen through a leaked backup, SQL injection, or an insider, so the goal of password storage is to make a stolen file useless for offline guessing. Storing passwords in plaintext fails immediately: every account is compromised the moment the file leaks. Encrypting passwords is the wrong tool, because encryption is reversible and an attacker who steals both the file and the decryption key recovers every password. Passwords must be hashed, not encrypted.

A fast hash alone, such as SHA-256 or MD5, is not enough. An attacker with a GPU can test billions of guesses per second, identical passwords produce identical hashes and reveal which users share a password, and rainbow tables let an attacker precompute hashes of common passwords once and reuse the tables against every database.

A salt fixes sharing and precomputation. A salt is a unique, random, non-secret value stored in the clear next to each hash, and the system hashes the salt together with the password rather than the password alone. Two users with the same password then produce completely different records, and the attacker must recompute every guess for every user separately. A 128-bit random salt per user is a common choice. The salt does not slow down brute force, because each guess is still one fast hash.

A slow, work-factor-tunable key derivation function (KDF) makes each guess expensive for the attacker while staying cheap enough, roughly 250 milliseconds to 1 second, for a single legitimate login. Four KDFs are in use:

- **Argon2id:** the preferred choice for new systems, memory-hard and tunable.
- **scrypt:** memory-hard, consuming about 16 MiB per guess at the common N=2^14, r=8, p=1 parameters, which defeats massively parallel GPU cracking.
- **bcrypt:** the long-standing standard, based on the Blowfish cipher.
- **PBKDF2:** the accepted minimum, standardized and widely built in, used with a high iteration count such as 600,000 iterations of HMAC-SHA256.

Real systems store everything needed to recompute the hash in one self-describing string: `$algorithm$cost$salt$hash`. For example, `/etc/shadow` on Linux uses `$6$salt$hash` for sha512crypt, and a modern Argon2id record looks like `$argon2id$v=19$m=65536,t=3,p=4$salt$hash`. The algorithm, work factor, salt, and hash travel together, so the verifier needs no separate configuration.

Verification must compare the recomputed hash against the stored hash with a constant-time comparison, such as `hmac.compare_digest` in Python, so the comparison time does not leak how many characters were correct. Good storage still cannot fix a weak password: a dictionary attack cracks `qwerty7` regardless of the KDF, only slower. Password complexity policy, rate limiting, account lockout, and MFA must accompany good storage.

Authentication credentials such as passwords should be securely stored and maintained. A password vault, also called a password manager, is a software application that stores, manages, and secures a user's passwords so the user can select complex passwords without memorizing them. For example, 1Password and LastPass are popular password managers.

#### **Account Types**

A computer user is associated with a computer account, and the account contains information about the user and the user's permissions to computer resources. Five types of accounts exist:

- **User account:** an account assigned to an individual who wants to access a computer resource, also called an end user account.
- **Privileged account:** an account assigned to a system administrator, also called an administrative or root account, with full authorization and control over a system. A privileged account administers and maintains the system's resources and users. For example, the root account in Linux/Unix and the administrator account in Windows are privileged accounts.
- **Shared account:** an account accessed by more than one user, also called a generic or group account, used by a group of users who need access to the same resources. All users in the group share the account's credentials.
- **Guest account:** an account assigned to a temporary user, with limited privileges.
- **Service account:** an account assigned to an application or service, not used for interactive login.

### 12. Accounts: Controls and Maintenance

#### **Account Controls**

Access to a user account can be controlled based on the physical characteristics of the user's device. Geolocation uses location technologies such as GPS coordinates or an IP address to identify the location of an electronic device. Geotagging is the process of adding geographical identification to a device. Geotagging can restrict system access to devices at a specific GPS coordinate or with a specific IP address.

Geofencing is the practice of using GPS or radio frequency identification (RFID) to define a virtual perimeter for a geographic boundary. A geofenced area is a predetermined GPS-based area from which a user can log in to an account.

A device's geolocation can also block logins from implausible locations. Impossible travel time is a metric a system uses to deny a login attempt from a location the user could not have reached in the elapsed time since the previous login. For example, a login attempt from Los Angeles at 9:05 AM is denied if the user's previous login was from Boston at 9:00 AM.

A device's network location is the device's connection point to a network, determined by the device's media access control (MAC) address or IP address. For example, a user account can be configured to accept logins only from the IP address assigned to the user's laptop.

Time-of-day restrictions specify when a user can log in to an account, preventing access outside work hours. For example, a user may be allowed to log in only Monday through Friday from 8:00 AM to 5:00 PM. Time-based login enables a user to log in at a specific time.

#### **Account Maintenance**

Account controls are necessary but not sufficient to secure a system. An account audit establishes whether account controls are properly implemented and effective. An account audit verifies that each user has access rights based on the user's needs and the system's security policies, and is performed periodically, either manually or automatically.

An account that is temporarily unneeded or no longer secure can be disabled. A disabled account exists in the system but is not accessible by the account owner. A user account can also be automatically locked out to prevent or slow down a brute-force attack against the user's credentials. An account lockout disables an account when the number of incorrect login attempts exceeds a predefined threshold. A locked account can be re-enabled manually by an account administrator or automatically after a period of time called the lockout duration.

#### **Attestation**

Attestation is the process of verifying a device's hardware and software configurations to ensure trustworthiness. Attestation enhances account security by allowing only compliant devices to access network resources, and can integrate with geofencing and time-based restrictions to add a security layer that blocks potentially compromised devices from sensitive data. For example, before a laptop joins an organization's network, attestation verifies that the laptop's antivirus software is up to date and the firewall is enabled.

### 13. Access Control Models: DAC and MAC

#### **Discretionary Access Control (DAC)**

Discretionary access control (DAC) is an access control model in which access to an object is at the discretion of the object's owner. The owner defines access to the object based on a subject's need for that access. DAC is implemented with an access control list (ACL), which defines a subject's access type to an object. For example, under DAC a user's access to a file is at the file owner's discretion: the owner can grant the user write permission to the file or deny write permission. DAC is the default access control model in most operating systems, including Linux/Unix and Windows.

#### **Mandatory Access Control (MAC)**

Mandatory access control (MAC) is an access control model in which access to an object is mandated by a set of rules and classification labels. A classification label represents a security domain, a collection of subjects and objects that share a common security policy. Each subject is assigned a clearance level, and each object is assigned a classification label. A MAC security policy defines a subject's access privileges to an object based on the subject's clearance level and the object's classification label. A central authority assigns clearance levels and classification labels and defines the security policies. For example, a user with a "secret" clearance level is denied access to a file with a "top secret" classification label.

MAC is the most restrictive access control model and is typically used in government and military settings where data confidentiality is a high priority.

#### **Comparing DAC and MAC**

DAC and MAC differ in who controls access: the object's owner in DAC, a central authority in MAC. This single difference drives every other property of the two models.

| Property | DAC | MAC |
| --- | --- | --- |
| Who decides access | The object's owner | A central authority |
| Mechanism | Access control list (ACL) | Clearance levels and classification labels |
| Access based on | The subject's identity | The subject's clearance versus the object's classification |
| Owner can grant or override | Yes | No |
| Flexibility | High: the owner changes access at any time | Low: only the central authority changes policy |
| Restrictiveness | Least restrictive model | Most restrictive model |
| Typical use | Default in Linux/Unix and Windows | Government and military systems |

The two decision flows side by side:

```text
DAC decision flow (the owner controls the ACL):

  +--------------------------------------+
  | file owner sets the ACL on report:   |
  |   alice: read, write                 |
  |   bob:   read                        |
  +--------------------------------------+
            alice requests write
                    |
                    v
        ACL lookup: alice = read, write
                    |
                    v
                 PERMIT
  (the owner can edit the ACL at any time)
```

```text
MAC decision flow (the central authority controls labels):

  central authority assigns:
    alice     -> clearance: secret
    report    -> classification: top secret

            alice requests read
                    |
                    v
    policy check: clearance >= classification?
    is secret >= top secret?  no
                    |
                    v
                  DENY
  (neither alice nor the file's owner can override)
```

The same scenario shows the practical consequence. Under DAC, Alice owns a file and grants Bob write access with one ACL change, which suits collaboration but also means any malware running under Alice's account inherits Alice's full discretion over Alice's files. Under MAC, Alice's ownership of a file grants no special power: a user with secret clearance cannot read a top secret file, cannot copy top secret data into a secret file to leak the data downward, and cannot grant a colleague access regardless of ownership. MAC trades the owner's flexibility for the guarantee that a system-wide confidentiality policy holds for every subject and every object.

#### **SELinux**

To increase security for confidential data on Linux machines, the US National Security Agency (NSA) developed SELinux, a security architecture that supports MAC implementation on Linux. SELinux was released to the open source community in 2000 and is now included in the Linux kernel. SELinux is most commonly used on Red Hat Enterprise Linux (RHEL) and Fedora Linux, and has provided Android's MAC system since Android 4.3, released in 2013.

### 14. Access Control Models: Role, Rule, and Attribute-Based

#### **Role-Based Access Control (RBAC)**

Role-based access control (RBAC) is an access control model in which a subject's access rights to an object are based on the subject's role within a system. An RBAC role is a collection of objects and permissions that capture the requirements of a job function. A subject assigned to a role gains the access rights required to perform the role's job, and a subject can access an object only if the subject holds a role granting that access. For example, a sales role captures the requirements for a salesperson, including access rights to a database and a computer. A user who works as a salesperson is assigned to the sales role and thereby gains access to the database and the computer.

RBAC is a non-discretionary access control model: access to an object is controlled by a central authority or administrator rather than by the object's owner.

Example: a hospital defines four roles, and each role bundles the permissions a job function needs:

| Role | Patient records | Prescriptions | Billing system | Appointment calendar |
| --- | --- | --- | --- | --- |
| Doctor | read, write | create | none | read, write |
| Nurse | read | none | none | read, write |
| Receptionist | none | none | none | read, write |
| Billing clerk | none | none | read, write | read |

A newly hired nurse receives the nurse role on day one and immediately holds exactly the permissions in the nurse row: read access to patient records and read-write access to the appointment calendar, with no access to prescriptions or billing. When the nurse transfers to the billing department, the administrator removes the nurse role and assigns the billing clerk role, and the nurse's old permissions disappear with the old role. No administrator ever edits a permission directly on a person: access changes only through role assignment.

#### **Attribute-Based Access Control (ABAC)**

Attribute-based access control (ABAC), also called policy-based access control (PBAC), is an access control model in which access to an object is based on attributes and on access control policies that define the allowable operations for a given attribute combination. ABAC is a non-discretionary access control model and uses three types of attributes:

- **Object attribute:** a characteristic of an object, such as name, creation date and time, sensitivity, and owner.
- **Subject attribute:** a characteristic of a subject, such as name, role, organization, and security clearance.
- **Environment attribute:** a characteristic of the access request's context, such as time of access and current threat levels.

Under ABAC, a subject is granted access rights to an object when a subject-object-environment attribute combination matches a defined access control policy. For example, a user's access to a file is denied if the combination of the user's security clearance, the file's creation time, and the current threat level does not match a defined policy.

Example: a company defines one ABAC policy for financial reports:

```text
Policy: permit read on a financial report when ALL conditions hold:
  subject.department = finance
  subject.clearance >= report.sensitivity
  environment.time is within business hours (Mon-Fri, 8:00 AM - 5:00 PM)
  environment.threat-level = normal
```

Two requests against this policy show the attribute combination at work:

- **Request 1, permitted:** Carol works in finance and holds a confidential clearance. Carol requests the quarterly report, labeled internal sensitivity, on Tuesday at 10:00 AM while the threat level reads normal. All four conditions hold, and the policy permits the read.
- **Request 2, denied:** Carol requests the same report on Saturday at 11:00 PM. The subject and object attributes still match, but the environment attribute fails: the time falls outside business hours. One failed condition denies the request.

The same Carol, the same report, and the same action produce opposite outcomes because the environment changed. RBAC cannot express this distinction without creating extra roles, while ABAC expresses the distinction in one policy.

#### **Rule-Based Access Control**

Rule-based access control is an access control model in which access to an object is based on a set of rules. A subject can access an object only when all conditions in a rule are met. A rule defines the circumstances under which a subject can access an object. For example, a rule may allow access from a specific IP address, restrict access to business hours only, or deny access from a mobile device. Rule-based access control is a non-discretionary model in which a central authority or administrator defines the rules.

Rule-based access control is commonly implemented in firewalls through access control lists. A firewall uses a set of rules to filter network traffic. For example, a firewall rule may allow HTTPS traffic on TCP port 443 or deny SSH traffic on TCP port 22.

Example: a company firewall protects a web server with five rules, evaluated top to bottom with a first match deciding the outcome:

```text
Rule  Source            Destination      Protocol/Port     Action
1     10.1.2.0/24       web server       TCP 22 (SSH)      allow   (admin network)
2     any               web server       TCP 22 (SSH)      deny    (all others)
3     any               web server       TCP 443 (HTTPS)   allow
4     any               web server       TCP 80 (HTTP)     deny    (force HTTPS)
5     any               any              any               deny    (default deny)
```

Three packets show the rules in action:

- **Packet 1:** an administrator at 10.1.2.15 opens SSH to the web server. Rule 1 matches the source network, protocol, and port, and the firewall allows the connection.
- **Packet 2:** an external attacker at 203.0.113.9 opens SSH to the web server. Rule 1 fails on the source, rule 2 matches, and the firewall denies the connection.
- **Packet 3:** a visitor opens HTTP on TCP port 80 to the web server. Rules 1 through 3 fail to match, rule 4 matches, and the firewall denies the connection, forcing the visitor to retry over HTTPS.

No rule names a person. Every decision follows from conditions on the packet: source, destination, protocol, and port. Rule 5, the default deny, blocks anything no earlier rule matched, so an unlisted protocol such as Telnet on TCP port 23 never reaches the server.

### 15. Access Control: Filesystem Permissions, PAM, and Conditional Access

#### **Filesystem Permissions**

Filesystem permissions control which users, groups, or services can perform an action on a file: reading, writing, modifying, or executing the file. Improper or insecure file permissions can be leveraged by a user to perform unauthorized actions on a file.

Filesystem permissions are implemented differently in each operating system. In Linux/Unix, a file has read, write, and execute permissions, denoted by a set of three characters: r for read, w for write, and x for execute. If an action is not permitted, the corresponding letter is replaced by -. For example, a file permission of r-x means the user can read and execute the file but not write to the file, and rw- means the user can read and write the file but not execute the file.

Linux/Unix defines three permission classes: owner, group, and other. The ls command with the -l option displays file permissions for all three classes. The first character of the ls -l output shows the file type and is unrelated to permissions. The remaining nine characters form three sets of three: the first set shows owner permissions, the second set shows group permissions, and the third set shows other permissions. For example, -r-xrwxr-x means the owner class has read and execute permissions, the group class has read, write, and execute permissions, and the other class has read and execute permissions. Similarly, -rw-rw---x means the owner and group classes have read and write permissions, and the other class has execute permission only.

In Windows, file permissions are configured in the Security tab of a file's properties. Five permissions exist for a file:

- **Full control:** read, write, modify, and delete the file, and change the file's permissions.
- **Modify:** read and modify the file.
- **Read & execute:** read and execute the file.
- **Read:** read the file.
- **Write:** write to the file. Only Full control includes changing a file's permissions.

#### **Privileged Access Management (PAM)**

Privileged access management (PAM), also called privileged identity management (PIM), is the set of controls, tools, and processes for managing, securing, and monitoring privileged accounts. A privileged account has elevated access rights in a system. For example, the administrator account in Windows and the root account in Linux/Unix are superuser privileged accounts.

A privileged account holds greater security permissions than any other account type and therefore has the potential to cause more harm. A privileged account may have authorization to override security controls and to perform tasks such as configuring networks, shutting down systems, and provisioning accounts. Managing privileged accounts should enforce the principle of least privilege, which states that a user should be granted only the minimum access rights required to perform the user's job.

#### **Conditional Access**

Conditional access is access control based on predefined access criteria or conditions. Access criteria can include a subject's geolocation, device type, or IP address, or a device's security state, such as having an up-to-date antivirus signature file or the latest security patch. The resulting access control decision can block, limit, or allow access to an object, require the subject to complete multi-factor authentication, or request installation of a critical security patch on the device.

Conditional access commonly enforces security policies. A system that supports conditional access analyzes real-time conditions and the system's security policies to make an enforcement decision. For example, conditional access can enforce a policy that requires a user to complete two-factor authentication before granting access to an accounting application. Microsoft Entra ID, formerly Azure Active Directory, and Microsoft Intune support conditional access.

## Summary

This `CHAP`ter defined identity and access management through its three objectives: identification, authentication, and authorization. The `CHAP`ter described the five authentication factors, one-time passwords (TOTP and HOTP), hardware security through TPM, HSM, and secure enclaves, and biometric authentication with the FRR, FAR, and CER error metrics. The `CHAP`ter examined the authentication protocols `PAP`, `CHAP`, Kerberos, `EAP` with IEEE 802.1X, RADIUS, and TACACS+, and the Internet standards SAML, OpenID, and OAuth. The `CHAP`ter closed with account policies, password storage through hashing, salting, and slow KDFs, account types, account controls and maintenance, the five access control models (DAC, MAC, RBAC, ABAC, and rule-based), filesystem permissions, privileged access management, and conditional access.

## Useful References and Resources

- NIST Special Publication 800-63B, "Digital Identity Guidelines: Authentication and Lifecycle Management," covering authentication factors, identity proofing, and authenticator requirements.
- RFC 6238, "TOTP: Time-Based One-Time Password Algorithm," and RFC 4226, "HOTP: An HMAC-Based One-Time Password Algorithm," the specifications of the two OTP types.
- NIST FIPS 140-3, "Security Requirements for Cryptographic Modules," the current publication in the FIPS-140 series used to certify TPMs and HSMs.
- RFC 1994, "PPP Challenge Handshake Authentication Protocol (`CHAP`)," and RFC 1334, "PPP Authentication Protocols," covering `CHAP` and `PAP`.
- RFC 4120, "The Kerberos Network Authentication Service (V5)," the specification of the Kerberos ticket-based protocol.
- RFC 3748, "Extensible Authentication Protocol (EAP)," and IEEE 802.1X-2020, "Port-Based Network Access Control," covering the `EAP` framework and the supplicant, authenticator, and authentication server roles.
- RFC 2865, "Remote Authentication Dial In User Service (RADIUS)," and RFC 8907, "The Terminal Access Controller Access-Control System Plus (TACACS+) Protocol," covering the two centralized AAA protocols.
- OASIS, "Assertions and Protocols for the OASIS Security Assertion Markup Language (SAML) V2.0," the SAML assertion specification.
- OpenID Foundation, "OpenID Connect Core 1.0," and RFC 6749, "The OAuth 2.0 Authorization Framework," covering federated authentication and delegated authorization.
- NIST INCITS 359, "Role Based Access Control," the standard specification of the RBAC model.
- RFC 8018, "PKCS #5: Password-Based Cryptography Specification Version 2.1," the specification of PBKDF2, and the OWASP Password Storage Cheat Sheet, current guidance on Argon2id, scrypt, bcrypt, salting, and work factors.
