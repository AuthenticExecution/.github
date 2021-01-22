# Tutorial: develop an Authentic Execution application

This brief tutorial will help you creating a distributed application from scratch using our Authentic Execution framework.

## Prerequisites

- The framework is installed on your local machine (guide [here](install-from-sources.md)), OR
- You use the `gianlu33/reactive-tools` Docker image (more info [here](https://github.com/gianlu33/authentic-execution#getting-started-with-docker))

## Examples of Authentic Execution applications

You might want to learn by having a look at other examples. If so, here some links:

- [Demo of this repository](../demo)
- [Prototype for a smart irrigation system](https://github.com/gianlu33/thesis/tree/master/prototype)
- [reactive-tools testing examples](https://github.com/gianlu33/reactive-tools/tree/master/examples)

## Initial stuff

- You have to create a folder (the _workspace_) which will contain all the modules and the deployment descriptors

## Develop an SGX or native module

### Create the module

Run `cargo new --lib <name>` to create a new Rust library

- **Note**: this is not really a library; a `main.rs` file will be added later during the building phase
- All the special annotations described below (inputs, outputs, requests, handlers) **MUST** be declared in the `lib.rs` file. Other functions or static data can be declared in other files.

### Declare entry points

An _entry point_ of a module is an interface for any entities capable of reaching the module. Therefore, anyone can call entry points of a module, hence you must be aware of the security implications.

Three entry points are available by default on each SGX module:

- `set_key`: called by the deployer to establish new connections. The payload is encrypted and authenticated using the **module key**
- `handle_input`: called by the Event Manager to deliver an event of an _output-input_ connection. The payload is encrypted and authenticated using the **connection key**
- `handle_handler`: called by the Event Manager to deliver an event of a _request-handler_ connection. The payload is encrypted and authenticated using the **connection key**

**Custom entry points**

To declare a user-defined entry point the following syntax is used:

```rust
//@ sm_entry
pub fn entry_point_name(data : &[u8]) -> ResultMessage {
    // body of the function
    
    // return value, e.g., everything went good, no payload
    success(None)
}
```

Notice the `//@ sm_entry` annotation above the function and the signature of it: different signatures (e.g., parameters and return value) are not allowed.

A [`ResultMessage`](https://github.com/gianlu33/rust-sgx-libs/blob/master/reactive_net/src/result_message.rs) inglobates a `ResultCode` and optional payload. The Authentic Execution framework provides two helper functions that can be used to create a `ResultMessage`:

```rust
pub fn success(data : Option<Vec<u8>>) -> ResultMessage {
    ResultMessage::new(ResultCode::Ok, data)
}

pub fn failure(code : ResultCode, data : Option<Vec<u8>>) -> ResultMessage {
    ResultMessage::new(code, data)
}
```

### Declare outputs and inputs

_output-input_ connections are used to trigger **asynchronous** events between modules. This kind of connection does not expect any value in return, therefore the module who triggers the output does not wait for an acknowledgements from the destination module. 

Outputs and inputs have a **n:n relationship**: an output can trigger multiple inputs and the same input can be triggered by multiple outputs.

**Outputs**

Outputs only needs to be declared using a special `//@ sm_output(name)` annotation. The corresponding function will be automatically generated during the building phase.

To call an output within another function (which may also be an entry point or an input), you need to pass a slice of a byte array as parameter:

```rust
//@ sm_output(output_name)

fn func() {
    let v : Vec<u8> = vec!(1,2,3);
    
    // call output_name borrowing the v array
	output_name(&v);
    
    // call output_name passing an empty array
    output_name(&[]);
}

```

The payload provided will be encrypted and authenticated before sending the event to the EM.

**Inputs**

Inputs are user-defined functions declared with the `//@ sm_input` annotation. The signature of inputs is fixed:

```rust
//@sm_input
pub fn input_name(data : &[u8]) {
    // body
}
```

The `data` payload is in clear, i.e., is decrypted before the input function is called.

### Declare requests and handlers

_request-handler_ connections are used to trigger **synchronous** events between modules. Unlike `output-input` connections, these events expect a value in return, which is an array of bytes. The source module, after triggering a request, waits for a response from the destination module. Only after the response is received the execution of the current function resumes.

Requests and handlers have a **1:n relationship**: a request can only trigger a single handler, but the same handler can serve multiple requests.

**Requests**

To declare a request, we use a special `//@ sm_request(name)` annotation. The corresponding function will be automatically generated during the building phase.

To call a request, the procedure is the same as for outputs. However, in this case the function returns a `Result<Vec<u8>, Error>`:

```rust
//@ sm_request(request_name)

fn func() {
    let response = match request_name(&[]) {
        Ok(payload) => payload,
        Err(e)		=> {
            error!(&format!("Something went wrong: {}", e));
            return;
        }
    };
    
    // use the response payload
}
```

The `Error` enum is defined [here](https://github.com/gianlu33/rust-sgx-libs/blob/master/reactive_net/src/lib.rs).

**Handlers**

Handlers are user-defined functions declared with the `//@ sm_handler` annotation. The signature of handlers is fixed: unlike inputs, handlers return a `Vec` of bytes.

```rust
//@ sm_handler
pub fn handler_name(data : &[u8]) -> Vec<u8> {
    // body
    
    // return value
    vec!(1,2,3)
}
```

The `data` payload is in clear, i.e., is decrypted before the input function is called.

### Other things

- The Authentic Execution framework provides some macros for logging: `debug!`, `info!`, `error!`, `warning!`
  - All of them expect only a `&str` slice as parameter
  - For multiple parameters, you can use the `format!` macro: e.g.`info!(&format!("{} {}", par1, par2))` 
  - The `debug!` macro does not print anything by default. Debug prints can be enabled by compiling the module with the `debug_prints` feature
    - In the deployment descriptor, add `"features": ["debug_prints"]` to the module definition
- For more information on the code generator, you can check the [rust-sgx-gen stubs](https://github.com/gianlu33/rust-sgx-gen/tree/master/rustsgxgen/stubs).
- You are free to include any dependencies you want if you need them. Notice, however, that for SGX some dependencies, as well as some parts of the Rust standard library, are not available
  - Check [this](https://edp.fortanix.com/docs/tasks/dependencies/) and [this](https://edp.fortanix.com/docs/concepts/rust-std/) link for more information

## Develop a Sancus module

### Create the module

Every Sancus module consists of a set of C source files. In the deployment descriptor, we declare all these files on the definition of the module. Normally, you just need a single source file.

### Declare entry points

An _entry point_ of a module is an interface for any entities capable of reaching the module. Therefore, anyone can call entry points of a module, hence you must be aware of the security implications.

Three entry points are available by default on each Sancus module:

- `set_key`: called by the deployer to establish new connections. The payload is encrypted and authenticated using the **module key**
- `handle_input`: called by the Event Manager to deliver an event of an _output-input_ connection. The payload is encrypted and authenticated using the **connection key**

**Custom entry points**

To declare a user-defined entry point the following syntax is used:

```c
SM_ENTRY(module_name) void entry_point(uint8_t* input_data, size_t len) {
    
}
```

Notice the `SM_ENTRY` macro on the function: we define the name of the module between brackets. The Sancus entry points accept a byte array as first parameter and the size of the byte array as second parameter. No return value is expected.

### Declare outputs and inputs

_output-input_ connections are used to trigger **asynchronous** events between modules. This kind of connection does not expect any value in return, therefore the module who triggers the output does not wait for an acknowledgements from the destination module. 

Outputs and inputs have a **n:n relationship**: an output can trigger multiple inputs and the same input can be triggered by multiple outputs.

**Outputs**

Outputs only needs to be declared using a special `SM_OUTPUT` macro. The corresponding function will be automatically generated during the building phase.

To call an output within another function (which may also be an entry point or an input), you need to pass a byte array and its length as parameters:

```c
SM_OUTPUT(module_name, output_name);

void SM_FUNC(module_name) func() {
    int var = 1;
    
    // Call the output passing a byte array and its length as parameters
    output_name((unsigned char *) &var, sizeof(var));
}
```

The payload provided will be encrypted and authenticated before sending the event to the EM.

**Inputs**

Inputs are user-defined functions declared with the `SM_INPUT` macro. The signature of inputs is fixed:

```rust
SM_INPUT(module_name, input_name, data, len) {
  // body
}
```

The `data` byte array has length `len` and its content is in clear, i.e., is decrypted before the input function is called.

### Declare requests and handlers

Currently, Sancus does not support request-handler connections.

### Other things

- We _strongly_ recommend to learn the Sancus syntax before building Sancus modules. Data and functions are protected in the enclave using macros, such as `SM_DATA` and `SM_FUNC`. [Here](https://distrinet.cs.kuleuven.be/software/sancus/examples.php) you can find some examples.

## Write the deployment descriptor

The _deployment descriptor_ is a JSON file that is used by `reactive-tools` to deploy the application.

It is composed of four main sections:

- `"nodes"`: list of all the nodes where modules will be deployed to.
- `"modules"`: list of all the modules that are part of the application to deploy
- `"connections"`: list of all the secure connections between modules, and between deployer and modules. This list includes _output-input_ and _request-handler_ connections (_optional_)
- `"periodic-events"`: list of events that will be triggered periodically to call a specific entry point of a module. (_optional_)

### Nodes

All nodes have to be declared with the following information:

- `"type"`: indicates the type of the node (e.g., a Sancus node or a SGX node)
- `"name"`: unique name of the node. Used as a reference for the other sections
- `"ip_address"`: IP address of the node.
- `"reactive_port"`: port used by the Event Manager to listen for events

**Note**: certain types of node might need additional fields.

**Declare a SGX node**

```json
{
    "type": "sgx",
    "name": "node1",
    "ip_address": "127.0.0.1",
    "reactive_port": 5000
}
```

**Declare a native node**

```json
{
    "type": "native",
    "name": "node1",
    "ip_address": "127.0.0.1",
    "reactive_port": 5000
}
```

**Declare a Sancus node**

```json
{
    "type": "sancus",
    "name": "node1",
    "ip_address": "127.0.0.1",
    "vendor_id": 4660,
    "vendor_key": "0b7bf3ae40880a8be430d0da34fb76f0",
    "reactive_port": 6000
}
```

- `"vendor_id"`: ID assigned to a specific vendor. Used to derive the vendor key given a node's Master Key.
- `"vendor_key"`: Key of the vendor. It is the result of a derivation function taking as input the node's Master Key and the vendor ID.

To generate a vendor key, read the following [README](https://github.com/gianlu33/authentic-execution/tree/master/utils/sancus#sancus-images)

For more information about vendor ID and key, read the [Sancus paper](https://distrinet.cs.kuleuven.be/software/sancus/publications/tops17.pdf)

### Modules

All modules have to be declared with the following information:

- `"type"`: indicates the type of the module
- `"name"`: unique name of the module. Used as a reference for the other sections. For SGX and native modules, this **must** match with the name of the folder created with `cargo new` (see above)
- `"node"`: the name of the node where we want to deploy the module. Must be of the same type!

**Note**: certain types of node might need additional fields.

**Declare a SGX module**

```json
{
    "type": "sgx",
    "name": "module1",
    "node": "node1",
    "vendor_key": "../vendor/private_key.pem",
    "ra_settings": "../vendor/settings.json"
}
```

- `"vendor_key"`: path to the vendor's RSA private key used to sign the SGX module
- `"ra_settings"`: path to the vendor's settings file including the API keys for the authentication to the Intel Attestation Server.

For more information about how to generate these files, see [here](../vendor/README.md)

**Declare a native module**

```json
{
    "type": "native",
    "name": "module1",
    "node": "node1"
}
```

**Declare a Sancus module**

```json
{
    "type": "sancus",
    "name": "module1",
    "files": ["source_file1.c"],
    "node": "node1"
}
```

- `"files"`: list of all the source files of the module

**Declare already deployed modules **

You can declare modules that have already been deployed previously. This might be useful e.g. if we want to establish a connection between a module already deployed and a module that has to be deployed. To do so, you **must** provide at least the following additional fields (to add to the previous ones):

```json
{
    "deployed": true,
    "id": 1,
    "key": "89bfaebddcec42d56ee42138afe1951d"
}
```

- `"deployed"`: boolean to declare that the module has been already deployed
- `"id"`: ID of the module, assigned during the previous deployment
- `"key"`: module's key in hexadecimal value, established during the previous deployment

**IMPORTANT:** this feature has not been tested for SGX and native modules and it is really likely that something would not work. For Sancus modules, declaring already deployed modules is useful for the Secure I/O functionality (e.g., `led_driver` in the demo is already included in the `reactive.elf` binary loaded in the Sancus device). Hence, for Sancus this has been tested and it is working, however we still need to specify the integer IDs of any outputs/inputs we want to connect to the deployed modules, instead of their names (look at the [deployment descriptor](../demo/complete.json)). **Currently, deploying new modules after the first deployment will not work. For the moment, always deploy on a "clean" environment.**

### Connections

We have different kinds of connection:

- _output-input_ connections
- _request-handler_ connections
- _Direct_ connections, from the deployer to a input/handler of a module

For each connection, we declare a specific Crypto library to use, which **must** be supported by both parties involved in the connection. The supported crypto libraries for each architectures are the following:

- SGX: 
  - `"aes"`: hardware-implemented
  - `"spongent"`: [software-implemented](https://github.com/stenverbois/spongent-rs)
- Sancus:
  - `"spongent"`: hardware-implemented

Currently, to connect a Sancus and a SGX module we are forced to use SPONGENT as the crypto library. However, the sw implementation on the SGX side is really slow (a single encryption/decryption takes hundreds of milliseconds). This is the **bottleneck** of any system composed by both Sancus and SGX modules. Future work might remove this limitation, e.g., by providing hw support for AES on the Sancus architecture.

**Declare a output-input connection**

```json
{
    "from_module": "button_driver",
    "from_output": "button_pressed",
    "to_module": "controller",
    "to_input": "button_pressed",
    "encryption": "spongent"
}
```

- `"from_module"`: source module
- `"from_output"`: name of the output that will trigger events associated to this connection
- `"to_module"`: destination module
- `"to_input"`: name of the input that will be called as a result of the output event
- `"encryption"`: crypto library to use

**Declare a request-handler connection**

```json
{
    "from_module": "webserver",
    "from_request": "get_presses",
    "to_module": "db",
    "to_handler": "get_presses",
    "encryption": "aes"
}
```

The only difference relies on the names of the `"from_request"` and `"to_handler"` fields.

**Declare a direct connection**

```json
{
    "name": "init-server",
    "direct": true,
    "to_module": "webserver",
    "to_input": "init",
    "encryption": "aes"
}
```

A direct connection is characterized by the presence of the `"direct"` field, set to `true`. Optionally, a name can be defined (as any other connections), to simplify for a deployer the interaction with the system after deployment. If not defined, a default name will be assigned.

### Periodic events

Periodic events are used to trigger the execution of periodic tasks (e.g., a sensor reading). If defined, a secondary thread on the EM where the module resides will send an event with the frequency specified by the developer, to trigger a specified entry point.

```json
{
    "module" : "controller",
    "entry" : "request_data",
    "frequency" : 1000
}
```

- `"module"`: name of the module that contains the entry point we want to call
- `"entry"`: name of the entry point to call
- `"frequency"`: periodicity of the event generation (in milliseconds)

**DISCLAIMER**: this is an **untrusted** feature used to simplify periodic operations such as sensor readings. The deployer has no guarantee that the requested entry point will be ever called, and with the specified frequency. Additionally, anyone on the system can call the same entry point, as the Authentic Execution framework does not provide by default any security guarantees for user-defined entry points.

**NOTE**: By default, this feature is disabled on the nodes. To enable the management of periodic events on a SGX or native node, the corresponding event manager must be launched with the environment variable: `EM_PERIODIC_TASKS=true`. Concerning Sancus nodes, limitations on the current implementation of the embedded OS does not allow to have timers that work properly. Hence, this feature is not currently available for Sancus nodes.

## Deployment and beyond

For more information about the commands to deploy and interact with the system, you can:

- Run the help commands of reactive-tools 
  - `reactive-tools -h` or 
  - `reactive-tools {build,deploy,call,output,request} -h`

- Check the [Makefile](../Makefile) in the root folder of this repository
- Read the README in the [reactive-tools](https://github.com/gianlu33/reactive-tools) repository