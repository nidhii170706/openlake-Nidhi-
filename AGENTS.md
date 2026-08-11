# AGENTS.md

## Project Overview

High performance storage engine for KV cache offloading and LLM Pretraining

# OpenLake KV-cache Deployment Guidance

Inspect the target host or GPU cluster and select the implemented OpenLake
KV-cache offloading path when configuring, deploying, validating, or
troubleshooting OpenLake across local, RDMA, DCT, UCX, NVIDIA, AMD,
single-node, or multi-node environments.

Determine the deployment from observed system capabilities and the current
OpenLake source. Do not select a mode from memory alone.

This guidance applies only to external KV-cache offloading. OpenLake object
storage is a separate product mode with different runtime, storage, and
configuration semantics. Do not use this guidance to configure object storage.

## 1. Establish the workload

Determine:

- Is vLLM the inference engine?
- Is the deployment single-node or multi-node?
- How many GPUs and GPU nodes are involved?
- What model, KV-cache format, and block size will be used?
- What capacity, latency, and throughput targets apply?

## 2. Inspect the target system

Use non-mutating commands where access is authorized:

```bash
uname -a
lscpu
lspci -nnk
ip -brief link
numactl --hardware
nvidia-smi
rocm-smi
ibv_devices
ibv_devinfo
rdma link
ucx_info -d
```

Record:

- Operating system and kernel
- CPU and NUMA topology
- GPU vendor and model
- GPU count per node
- NIC vendor, model, port, and link state
- InfiniBand, RoCE, or Ethernet
- RDMA device name
- UCX version and available transports
- CUDA or ROCm runtime
- Node count and addresses
- Whether the OpenLake binary includes the `rdma` feature

Require evidence from system commands. Do not infer capabilities from a product
name or configuration value alone.

## 3. Trace the current implementation

Treat source code as authoritative. Inspect these files before recommending a
path:

- `crates/openlake_server/src/config.rs`
  - KV configuration fields, required values, and rejected combinations.
- `crates/openlake_server/src/main.rs`
  - Runtime dispatch from KV mode, transport, and RDMA backend.
- `crates/openlake_server/src/kv_runtime.rs`
  - Local/TCP, DCT, and UCX KV server implementations.
- `crates/openlake_kv_client/src/lib.rs`
  - Connector dispatch for `local`, `ucx`, or an RDMA device.
- `crates/openlake_kv_client/src/shm_local.rs`
  - Local POSIX shared-memory and GPU-copy behavior.
- `crates/openlake_kv_client/src/protocol.rs`
  - Direct-verbs/DCT client path.
- `crates/openlake_kv_client/src/ucx_protocol.rs`
  - UCX client path.
- `crates/openlake_io/src/ucx_shim.c`
  - UCX memory registration and transfer operations.
- `external/connectors/vllm/openlake_adapter.py`
  - vLLM KV-memory registration and connector configuration.
- `crates/openlake_server/configs/`
  - Checked-in KV configuration examples.

Use line-level code references in the recommendation.

If documentation and implementation disagree, explain the discrepancy. With
the user's authorization, create an issue at:

https://github.com/openlake-project/openlake/issues

Without authorization to create the issue, ask the user to report it there and
provide a ready-to-submit issue title and description.

## 4. Keep the configuration dimensions separate

Evaluate each dimension independently:

- Server mode: `kv`
- Server transport: `h2` or `rdma`
- RDMA backend: `dct` or `ucx`
- Connector device: `local`, `ucx`, or an RDMA device
- Topology: single-node or multi-node
- GPU platform: NVIDIA or AMD

Do not use "mode," "transport," "backend," and "connector device"
interchangeably.

## 5. Select the implemented path

Apply these initial routing rules and verify each one against the current code.

### Same-host local KV offload

Consider:

```toml
mode = "kv"
transport = "h2"
```

with:

```json
"openlake_device": "local"
```

Use this path when vLLM and OpenLake run on the same host.

The host KV slab is exposed through POSIX shared memory and mapped into the
client process. Inspect `shm_local.rs` to verify that its GPU-copy
implementation supports the observed GPU platform.

Do not use local shared memory as a cross-node transport.

### UCX KV offload

Consider:

```toml
mode = "kv"
transport = "rdma"

[rdma]
backend = "ucx"
```

with:

```json
"openlake_device": "ucx"
```

Require:

- Linux
- An RDMA-enabled OpenLake build
- A working UCX installation
- A compatible network transport
- Successful memory registration for the memory type used by vLLM

Inspect `ucx_protocol.rs`, `ucx_shim.c`, and the installed UCX capabilities
before recommending this path.

### Direct-verbs/DCT KV offload

Consider:

```toml
mode = "kv"
transport = "rdma"

[rdma]
backend = "dct"
dev_name = "<observed RDMA device>"
```

Configure the connector with the corresponding observed RDMA device:

```json
"openlake_device": "<observed RDMA device>"
```

Require:

- Linux
- An RDMA-enabled OpenLake build
- A DCT-capable RDMA device
- The required DCT configuration fields
- Successful registration of the vLLM KV-cache memory

Verify the device name and transport capabilities from the target system.

### Multi-node KV offload

Each KV server is standalone in its own server configuration. Verify the
current `config.rs` validation before constructing its `[[nodes]]` section.

Use:

- `kv_agents` for the ordered KV-server address list required by OpenLake.
- `openlake_nodes` in the vLLM connector configuration.
- A consistent node ordering across every connector.
- Unique `self_id` values corresponding to the ordered server list.

Do not automatically copy an object-storage cluster's `[[nodes]]` topology
into a KV-server configuration.

## 6. Validate the selected path

Run the smallest applicable validation:

1. Build OpenLake with the required features.
2. Start OpenLake with the selected configuration.
3. Confirm the selected KV runtime in server logs.
4. Initialize the native client with the selected connector device.
5. Allocate memory using the same platform and type as production.
6. Register the memory.
7. Attach the client to the KV server.
8. Put known KV blocks.
9. Read those blocks into a cleared destination.
10. Synchronize the GPU when applicable.
11. Compare the retrieved content byte-for-byte.
12. Repeat across nodes for a multi-node deployment.
13. Measure latency and throughput after correctness passes.

## 7. Report the decision

Return:

- Observed hardware and software facts
- Selected OpenLake KV path
- Exact server configuration
- Exact vLLM connector configuration
- Code locations implementing the selection
- Why alternative KV paths were rejected
- Commands used to inspect the system
- Correctness tests performed
- Remaining validation work
- Qualification level:
  - `validated`
  - `expected but unqualified`
  - `unsupported`

Do not deploy, restart, stop, or deallocate infrastructure unless the user
explicitly authorizes that action.
