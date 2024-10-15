Podman support
==============


## Usage

* `crc setup`
* `crc start`
* `eval $(crc podman-env)`
* `podman version` (macOS/Windows) or `podman-remote version` (Linux)


## Limitations

* It exposes the **rootless** podman socket of the virtual machine
* It still require to have the full OpenShift cluster running
* Bind mounts don't work.

* Ports are not automatically exposed on the host.
  * Workaround when using vsock network mode:
    * Expose a port: `curl  --unix-socket ~/.crc/crc-http.sock http:/unix/network/services/forwarder/expose -X POST -d '{"local":":8080","remote":"192.168.127.3:8080"}'`
    * Unexpose a port: `curl  --unix-socket ~/.crc/crc-http.sock http:/unix/network/services/forwarder/unexpose -X POST -d '{"local":":8080"}'`
