Apple Silicon support
=====================


M1 support uses `vfkit` which is a small binary wrapper which maps command line arguments to the API provided by the [macOS virtualization framework](https://developer.apple.com/documentation/virtualization). It does this using the go bindings provided by https://github.com/Code-Hex/vz

## Lifecycle
The main reason for needing this separate vfkit binary is that when creating VMs with macOS virtualization framework, their lifetime is tied to the process which created them. This is why we need a separate process which will stay alive as long as the VM is needed.

> [!NOTE]
> There is no separate machine driver for vfkit, it's integrated directly in crc codebase, similarly to what is done for Hyper-V.

> [!NOTE]
> Apple silicon support has been available since [CRC 2.4.1](https://github.com/crc-org/crc/releases/tag/v2.4.1).
