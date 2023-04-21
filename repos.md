# Description of the AuthenticExecution repositories

## Main repositories

![Flow](images/repos.svg)

The diagram shows the most important repositories that are part of the framework
and how they related to each other.

### reactive-tools

[reactive-tools](https://github.com/AuthenticExecution/reactive-tools) is the
core of the framework. It is a Python application that takes as input the
deployment descriptor and the source code of the modules, and automatically
deploys and configures the application described in the descriptor.

Each modules is built using the corresponding TEE toolchain.
- Sancus modules are compiled using
  [sancus-compiler](https://github.com/AuthenticExecution/sancus-compiler)
- SGX modules are compiled using the [Fortanix EDP
  toolchain](https://edp.fortanix.com/docs/)
- TrustZone modules are compiled using the [OP-TEE
  toolchain](https://optee.readthedocs.io/en/latest/)

### Event managers

Each TEE provides an implementation of the Event Manager, an untrusted
application that manages events between modules and performs other operations
such as loading new enclaves.

The implementation of the EMs is provided, along with auxiliary files, in the
following repositories:
- [event-manager-sgx](https://github.com/AuthenticExecution/event-manager-sgx)
- [event-manager-sancus](https://github.com/AuthenticExecution/event-manager-sancus)
- [event-manager-trustzone](https://github.com/AuthenticExecution/event-manager-trustzone)
  - The TrustZone Event Manager is installed with
    [optee-examples](https://github.com/AuthenticExecution/optee_examples)

### SGX-related repositories

**rust-sgx-gen**

[rust-sgx-gen](https://github.com/AuthenticExecution/rust-sgx-gen) is a Python
module that generates the Authentic Execution code for SGX modules. It takes as
input the source code as implemented by the developer (using the API to declare
outputs and inputs) and generate the necessary code to implement the logic for
sending/receiving events.

It also automates the Remote Attestation of SGX modules: in our framework, we
use EPID attestation using an external library called
[rust-sgx-remote-attestation](https://github.com/AuthenticExecution/rust-sgx-remote-attestation),
which also uses [rust-mbedtls](https://github.com/fortanix/rust-mbedtls) to
establish secure TLS channels between the enclave and the IAS, and the enclave
and the attester. Both these repositories have been forked and slightly
modified. The attestation flow needs communication with the AESM service in the
node where the enclave is running, which is mediated by
[aesm-client](https://github.com/AuthenticExecution/aesm-client); This service
needs to be deployed on each **physical** node, and SGX modules must be
configured in the deployment descriptor to communicate with their local AESM
client.
[example](https://github.com/AuthenticExecution/examples/blob/main/smart-home/descriptor.json#L8)

**rust-sgx-libs**

[rust-sgx-libs](https://github.com/AuthenticExecution/rust-sgx-libs) contains
two Rust libraries that include some utility functions for the enclaves. The
first, `reactive_net`, simply implements the network protocol of our framework.
The second, `reactive_crypto` provides crypto primitives for authenticated
encryption using libraries such as AES-GCM-128 and SPONGENT-128. The latter is
provided by the
[spongent-cpp-rs](https://github.com/AuthenticExecution/spongent-cpp-rs)
library.

### Sancus-related repositories

Sancus modules are built using the Sancus compiler
([sancus-compiler](https://github.com/AuthenticExecution/sancus-compiler)) and
they include a utility library called
[sancus-support](https://github.com/AuthenticExecution/sancus-support). Both of
them have been forked in this organization: the former has been modified to
remove some commits that are not supported in our current implementation, while
the latter only adds a bugfix.

### TrustZone-related repositories

We made slight changes to the OP-TEE code
([optee_os](https://github.com/AuthenticExecution/optee_os)) to hardcode to the
trusted OS a hardware-unique key (HUK) for the attestation of TrustZone modules.
Our changes contain the efforts from the
[Distrinet-TACOS](https://github.com/Distrinet-TACOS) group, which made it
possible to deploy OP-TEE on
[BD-SL-i.MX6](https://boundarydevices.com/product/bd-sl-i-mx6/) boards. We have
forked
[shared-secure-peripherals](https://github.com/AuthenticExecution/shared-secure-peripherals)
to update some repositories locations during the installation of OP-TEE.

[trustzone-event-manager](https://github.com/AuthenticExecution/event-manager-trustzone)
not only contains the event manager code, but also instructions to build and run
TrustZone nodes on both QEMU and the i.MX6 board.


[TZ-Code-Generator](https://github.com/AuthenticExecution/TZ-Code-Generator),
instead, has the same purpose as `rust-sgx-gen`, adding necessary stub code to
TrustZone modules.

## Other repositories

- [examples](https://github.com/AuthenticExecution/examples) provides some
  example deployments using [Docker](https://www.docker.com/) and [Docker
  Compose](https://docs.docker.com/compose/)
- [attestation-manager](https://github.com/AuthenticExecution/attestation-manager)
  is an optional component that can be used to attest all modules in a
  deployment. It is deployed as an SGX enclave itself, and its use is seamlessly
  integrated with `reactive-tools` thanks to a [command line
  interface](https://github.com/AuthenticExecution/attestation-manager-client).
  Check our examples to know more.
- [sgx-attester](https://github.com/AuthenticExecution/sgx-attester) is a
  commodity application to attest SGX enclaves, used internally by
  `reactive-tools`.
- [reactive-uart2ip](https://github.com/AuthenticExecution/reactive-uart2ip) is
  a TCP server that mediates the communication with a Sancus device using UART.
  It is used internally by `event-manager-sancus`.
- [env](https://github.com/AuthenticExecution/env) contains some additional
  information about SGX, as well as a Makefile to run the Docker images
- [build](https://github.com/AuthenticExecution/build) is forked from the OP-TEE
  codebase, adding a minor patch for the execution of OP-TEE on QEMUv7
- [reactive-net](https://github.com/AuthenticExecution/reactive-net) is Python
  analogous to the `reactive_net` Rust library in `rust-sgx-libs`
- [reactive-base](https://github.com/AuthenticExecution/reactive-base),
  [docker-sgx-base](https://github.com/AuthenticExecution/docker-sgx-base),
  [docker-sgx-tools](https://github.com/AuthenticExecution/docker-sgx-tools) are
  intermediate docker images that are used by other repositories in the project