Block traffic with `nwfilter`
=============================


When testing the bundle, we needed to block NTP traffic to test the cert recovery code. Turns out this can be done directly for the crc VM with libvirt on linux

```bash
$ cat drop-ntp.xml
<filter name='drop-ntp' chain='ipv4'>
  <rule action='drop' direction='out' >
    <ip protocol='udp' dstportstart='123'/>
  </rule>
  <rule action='drop' direction='in' priority='100'>
    <ip protocol='udp' srcportstart='123'/>
  </rule>
</filter>

$ virsh -c qemu:///system nwfilter-define drop-ntp.xml
$ virsh -c qemu:///system edit crc
```

Then, in crc XML definitions, a `<filterref>` element needs to be added:
```xml
<domain>
  ...
  <devices>
  ...
    <interface type='network'>
      ...
      <filterref filter='drop-ntp'/>
    </interface>
```

This filter can be applied dynamically to a running VM using `virt-xml`:

```bash
$ virt-xml -c qemu:///system crc --edit --update --network filterref.filter='drop-ntp'
```

With that in place, we can run the cluster "in the future" by changing this in the domain xml:
```xml
<clock offset='variable' adjustment='30000' basis='utc'>
```

With 'adjustment' being a value in seconds. I need to experiment a bit more with this, as to exercise the cert recovery code, we probably need to change the time on the host too, and with NTP blocked, the cluster will probably sync with the host time without needing any changes to that `<clock>` element.


A similar nwfilter rule can be used for http/https traffic, this is useful for proxy testing. If the proxy is running on ports 3128/3129, this filter will block most http/https traffic which is not going through the proxy:
```xml
<filter name='drop-http-https' chain='ipv4'>
  <rule action='drop' direction='out' >
    <ip protocol='tcp' dstportstart='443'/>
  </rule>
  <rule action='drop' direction='in' priority='100'>
    <ip protocol='tcp' srcportstart='443'/>
  </rule>
  <rule action='drop' direction='out' >
    <ip protocol='tcp' dstportstart='80'/>
  </rule>
  <rule action='drop' direction='in' priority='100'>
    <ip protocol='tcp' srcportstart='80'/>
  </rule>
</filter>
```


## Reference
  - https://github.com/crc-org/crc/issues/1242#issuecomment-629698002
  - https://libvirt.org/formatnwfilter.html