Failed to add the user to the libvirt group in Fedora Silverblue
================================================================


If you are trying to use CRC on Fedora Silverblue, you might get following error.

    crc setup fails to add the user to the libvirt group.


This is a longstanding bug in Fedora Silverblue where the system groups in `/usr/lib/group` are not reflected in `/etc/groups`.
As a result, the libvirt group does exist after libvirt is installed, but the user cannot be added to said group via crc.


Users can manually work around this issue by copying the libvirt group info in /usr/lib/group to /etc/groups:

```bash
grep -E '^libvirt:' /usr/lib/group | sudo tee -a /etc/group
```

## References
  - https://docs.fedoraproject.org/en-US/fedora-silverblue/troubleshooting/#_unable_to_add_user_to_group

Thanks [@adambkaplan](https://github.com/adambkaplan) for creating issue and providing the workaround [#2402](https://github.com/code-ready/crc/issues/2402)
