# Security discussion

Brief considerations that summarize the security features of our framework. For further details, see the [research papers](https://github.com/gianlu33/authentic-execution#research).

## Deployer

- The deployer is the entity that "owns" the application to be deployed. As a consequence, a deploer is considered a **trusted** entity
- We assume to be trusted all the operations performed on the deployer's machine: building modules, generating encryption keys, storing sensitive data, etc.

## Nodes

- A node is generally untrusted, and it might run software from different vendors and for different purposes.
- Each nodes run an Event Manager, an **untrusted** software that manages all the events exchanged between modules, as well as the loading process of them.
- The nodes might not be owned by the deployer. For example, nodes can be VMs in the cloud.

## Modules

- Each module runs inside a Trusted Execution Environment (TEE). Therefore, code and data are protected w.r.t. confidentiality and integrity.
- Each module has a unique _Module Key_, known only by the deployer and the module itself.
  - For SGX: this key is established during the Remote Attestation process. It uses 128 bits of security.
  - For Sancus: this key is the result of a derivation function from the vendor key and the _identity_ of the module. It may use either 64 or 128 bits of security. For the demo, we use 128 bits.
- Sensitive communication between a module and the deployer is protected with the Module Key using authenticated encryption, using a 16-bit nonce to provide freshness and avoid replay attacks.

## Connections

- Each connection establishes a separate _secure communication channel_ using a unique _Connection Key_ known only by the two modules involved and the deployer. It is generated locally by the latter.
- Typically, a connection uses 128 bits of security. However, we might as well have 64-bit keys if one of the two modules involved runs in a Sancus node with 64 bits of security.
- Events belonging to a connection are protected with the Connection Key using authenticated encryption, using a 16-bit nonce to provide freshness and avoid replay attacks.

## Authenticated Encryption

- AES: we use [this](https://docs.rs/aes-gcm/0.3.0/aes_gcm/) AEAD cipher based on AES in Galois/Counter mode.
- SPONGENT: we use SPONGENT lightweight cryptographic library that is implemented in hardware on Sancus nodes.
  - [SW implementation for SGX modules](https://github.com/stenverbois/spongent-rs)

## Secure I/O

- Sancus' _Secure I/O_ feature can be used to allow a Sancus module to take exclusive use of the MMIO regions of I/O devices.
- This feature can be used to achieve **end-to-end** security, from high-end SGX to low-end Sancus nodes that include input and output devices.