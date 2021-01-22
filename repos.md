# Description of the AuthenticExecution repositories

## Main repositories

![Flow](images/repos.svg)

### reactive-tools

`reactive-tools` is the core of the framework. It is a Python application that takes as input the deployment descriptor and the source code of the modules, and automatically deploys and configures the application described in the descriptor.

Each modules is built using the corresponding TEE toolchain.
- Sancus modules are compiled using `sancus-compiler`
- SGX modules are compiled using the Fortanix EDP toolchain
- TrustZone modules are compiled using the OPTEE toolchain

### Event managers

Each TEE provides an implementation of the Event Manager, an untrusted application that manages events between modules and performs other operations such as loading new enclaves.

The implementation of the EMs is provided under the repositories `event_manager_{sgx,sancus,trustzone}`, along with auxiliary files.

### SGX-related repositories

**rust-sgx-gen**

`rust-sgx-gen` is a Python module that generates the Authentic Execution code for SGX modules. It takes as input the source code as implemented by the developer (using the API to declare outputs and inputs) and generate the necessary code to implement the logic for sending/receiving events.

It also automates the Remote Attestation of SGX modules: in our framework, we use EPID attestation using an external library called `rust-sgx-remote-attestation`, which also uses `rust-mbedtls` to establish secure TLS channels between the enclave and the IAS, and the enclave and the attester. Both these repositories have been forked and slightly modified.

**rust-sgx-libs**

This repository contains two Rust libraries that include some utility functions for the enclaves. The first, `reactive_net`, simply implements the network protocol of our framework. The second, `reactive_crypto` provides crypto primitives for authenticated encryption using libraries such as AES-GCM-128 and SPONGENT-128. The latter is provided by the `spongent-rs` library.

### Sancus-related repositories

Sancus modules are built using the Sancus compiler (`sancus-compiler`) and they include a utility library called `sancus-support`. Both of them have been forked in this organization: the former has been modified to remove some commits that are not supported in our current implementation, while the latter only adds a bugfix.

### TrustZone-related repositories

We made slight changes to the OPTEE code (`optee_os`) to hardcode to the trusted OS a hardware-unique key (HUK) for the attestation of TrustZone modules.

## Other repositories

- `examples` provides some example deployments using `docker` and `docker-compose`
- `reactive-uart2ip` is a TCP server that mediates the communication with a Sancus device using UART
- `env` contains some additional information about SGX, as well as a Makefile to run the Docker images
- `build` is forked from the OPTEE codebase, adding a minor patch for the execution of OPTEE on QEMUv7
- `reactive-net` is Python analogous to the `reactive_net` Rust library in `rust-sgx-libs`
- `reactive-base`, `docker-sgx-base`, `docker-sgx-tools` are intermediate docker images that are used by other repositories in the project