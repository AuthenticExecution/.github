# main

This repository contains documentation and links related to the Authentic
Execution framework.

## What is Authentic Execution?

We made a framework to allow developers to easily build and deploy distributed,
heterogeneous applications with strong **integrity** and **authenticity**
guarantees, as well as **confidentiality** of application data in transit, use,
and (optionally) at rest.

- Each component of the application (called _module_ or SM) runs in a **Trusted
  Execution Environment** (TEE) to enable _confidential computing_: code and
  sensitive data of the module are protected from any other software on the same
  machine.
- Communication channels between modules are **encrypted and authenticated**
  using symmetric keys: each single connection uses a different key, known only
  by the two modules of the connection and the deployer.
- Our framework is **heterogeneous**: currently, it supports
  [SGX](https://www.intel.com/content/www/us/en/architecture-and-technology/software-guard-extensions.html),
  [Sancus](https://distrinet.cs.kuleuven.be/software/sancus/) and [ARM
  TrustZone](https://developer.arm.com/ip-products/security-ip/trustzone). SGX
  is the most popular TEE, included in all recent Intel processors. Sancus is an
  open-source, embedded TEE designed for lightweight IoT systems. TrustZone is a
  security feature available in recent ARM processors (e.g., smartphones), on
  top of which OP-TEE provides a TEE implementation.
- The applications developed with this framework provide **end-to-end**
  **security** from high-end computation nodes to small low-end systems that
  include I/O devices (e.g., sensors, buttons, LCD screens, LEDs, etc.). Thanks
  to the Sancus **Secure I/O** functionality, in fact, we are able to secure the
  whole path from an input to an output device.
- Our framework provides an **abstraction layer** to the developer, who doesn't
  need to worry about the enclaved execution of the application nor the secure
  communication API. The only tasks for the developer are defining the logic of
  the application, declaring outputs/inputs of each module, and write a simple
  configuration file containing the description of the whole system (including
  the declaration of connections between modules).

## How does it work?

![Flow](images/flow.svg)

The user (or administrator) provides as input to our framework the source code
of each module of the application, as well as a configuration file that we call
_deployment descriptor_. The latter defines the topology of the application,
including the declaration of nodes, modules, and connections.

- Nodes are the physical (or virtualised) hardware resources that provide TEE
  capabilities, such as a SGX-enabled Intel machine or a Sancus board.
- Modules are the software components of the distributed application, of which
  the admin provides the source code.
- Connections are established between two modules, connecting the _output_ of a
  source module A to the _input_ of a destination module B. For example, a
  Sancus module `sensor` might be connected to an SGX module `aggregator` to
  send sensor readings.

### Developers API

Developers can write modules' code with extreme simplicity by only defining the
endpoints (outputs, inputs, entry points) of each module. For each TEE, we
provide a simple API in form of annotations or macros.

As an example, let's develop an application that implements the figure below.

![Flow](images/example.svg)

A Sancus module called `button_driver`, connected to a physical button using
Secure I/O, generates `button_pressed` outputs every time the button is pressed.
An SGX module called `db`, instead, stores the number of button presses, which
is incremented every time the input `increment_presses` is called. A connection
between `button_pressed` and `increment_presses` is therefore needed for
implementing the logic of the application.

**button_driver**

```c
#include <sancus/reactive.h>

#include <stdio.h>

SM_OUTPUT(button_driver, button_pressed);

SM_ISR(button_driver, num) {
    if (num != 2) {
        puts("Wrong IRQ");
        return;
    }

    puts("Button has been pressed!");
    button_pressed(NULL, 0);
}

SM_HANDLE_IRQ(button_driver, 2);
```

The code above implements the logic of `button_driver`. An output called
`button_pressed` is defined. An Interrupt Service Routine is implemented, which
should be called when the physical button is pressed. In the ISR code, the
output `button_pressed` is called and a new event will be generated (if the
output is connected to at least one input).

**db**

```rust
use std::sync::Mutex;

lazy_static! {
    static ref BUTTON_PRESSES: Mutex<u32> = {
        Mutex::new(0) // initially zero
    };
}

//@sm_input
pub fn increment_presses(_data : &[u8]) {
    let mut occ = BUTTON_PRESSES.lock().unwrap();
    *occ += 1;
}
```

The code above implements the code of `db`, which is a very simple logic to
store the number of button presses in a global variable. The input
`increment_presses` simply increments the number by 1. 

**Deployment descriptor**

```json
{
    "from_module": "button_driver",
    "from_output": "button_pressed",
    "to_module": "db",
    "to_input": "increment_presses",
    "encryption": "spongent"
}
```

To connect `button_driver` and `db`, a new connection is defined in the
deployment descriptor. As shown above, we specify the output and input to
connect, as well as the encryption library to use. In this case, we use
`spongent` as it is the only library supported by Sancus.

## Quick start

In the [examples](https://github.com/AuthenticExecution/examples) repository we
provide some examples of deployments that can be easily ran using `docker` and
`docker-compose`.

**NOTE**: the `native` version of the examples does not use TEE protection,
therefore they can be ran on simple Linux machines. This allows anyone to test
our framework and see how it works.

## Repositories

[Detailed information about the repositories of this organization](repos.md)

## Links

### Research Papers

- [Authentic Execution of Distributed Event-Driven Applications with a Small TCB](https://people.cs.kuleuven.be/~jantobias.muehlberg/stm17/), Noorman et al., STM'17.
- [End-to-End Security for Distributed Event-Driven Enclave Applications on Heterogeneous TEEs](https://dl.acm.org/doi/10.1145/3592607), Scopelliti et al., TOPS.

### Talks

- [FOSDEM 21: An Open-Source Framework for Developing Heterogeneous Distributed Enclave Applications](https://fosdem.org/2021/schedule/event/tee_sancus/)
- [CCS'21: POSTER: An Open-Source Framework for Developing Heterogeneous Distributed Enclave Applications.](https://bobspot.org/assets/docs/ccs-21/poster.pdf)

### Other Resources

- [Securing Smart Environments with Authentic Execution](https://distrinet.cs.kuleuven.be/software/sancus/publications/scopelliti2020.pdf), Scopelliti, Master's Thesis, 2020.
- [Sancus Website](https://distrinet.cs.kuleuven.be/software/sancus/index.php)

## Formal Definition and Proof of Security Properties

The STM'17 paper comes with an extended [appendix](secargument.pdf) in which we
focus on defining a simple reactive programming language for our architecture,
the formal semantics of that programming language, and the formal definition and
proof of the security properties of our approach.

## License

All the (non-forked) repositories in this organization are under MIT license.
Forked repositories retain the license of the original ones, with the exception
of `reactive-tools` that has been relicensed under MIT. External libraries and
other code have their own license.