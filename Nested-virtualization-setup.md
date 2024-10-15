Nested virtualization setup
===========================


> [!WARNING]
> Using nested virtualization with CRC is generally not supported.

In some cases nested virtualization works and can be useful for example for testing crc, such as on Windows.

Consider for example the following setup:

Ryzen 3600 with 32GiB RAM
```
vendor_id	: AuthenticAMD
cpu family	: 23
model		: 113
model name	: AMD Ryzen 5 3600 6-Core Processor
stepping	: 0
```

  - OS: RHEL8 (tested 8.3 and 8.4)
  - `libvirt-6.0.0-35.module+el8.4.0+10230+7a9b21e4.x86_64`
  - `qemu-kvm-4.2.0-48.module+el8.4.0+10368+630e803b.x86_64`

Most RHEL8 libvirt/qemu versions should work

Virtualization is enabled in the host's bios, and nested virt has to be explicitly enabled on the host through `options kvm_amd nested=1` in `/etc/modprobe.d/kvm.conf`

The VM is created using `virt-manager`, most settings are the default ones except for 14GB memory, 4 vcpus, and 120GB qcow2 disk. These numbers are not representative of the minimum required config, they are just the ones I picked to be sure the VM is correctly sized.

For nested virtualization to work, the VM CPU config must be set to "host-passthrough". This can be done before starting the Windows VM installation, or at a later time through the 'Details' window. The VM then needs to be shut off and started again for this setting to take effect.

After this, just install your Windows VM, make sure it's up to date, and you should be able to download and start crc as if on a real machine. Since it's a VM, you can take disk snapshots at interesting times, so that you can get back to that state later on.