Track TCP proxy connections
===========================


Create an image with this

`Containerfile`
```Dockerfile
FROM docker.io/library/centos:latest

RUN dnf install -y \
    bcc-tools \
    http://download.eng.bos.redhat.com/brewroot/vol/rhel-8/packages/kernel/4.18.0/147.3.1.el8_1/x86_64/kernel-devel-4.18.0-147.3.1.el8_1.x86_64.rpm \
    http://download.eng.bos.redhat.com/brewroot/vol/rhel-8/packages/kernel/4.18.0/147.3.1.el8_1/x86_64/kernel-headers-4.18.0-147.3.1.el8_1.x86_64.rpm \
    && dnf clean all \
    && rm -rf /var/cache/yum

ENTRYPOINT ["/usr/share/bcc/tools/tcpconnect"]
```

> [!NOTE]
> The `kernel-devel` and `kernel-headers` versions must exactly match the one used by the CRC bundle


Image creation and publishing

```bash
$ podman build -t bcc-tcpconnect -f Containerfile .
$ podman push localhost/bcc-tcpconnect quay.io/teuf/experiments:147.3.1.el8_1
```

> [!NOTE]
> The image is published to ensure the VM is able to download the image


Then after running `crc start`, you can run (possibly as soon as `ssh` is up in the VM):
```bash
$ ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i ~/.crc/machines/crc/id_rsa core@192.168.130.11 \
    sudo podman run --privileged -v /lib/modules:/lib/modules:ro \
    quay.io/teuf/experiments:4.18.0-147.3.1.el8_1
```
