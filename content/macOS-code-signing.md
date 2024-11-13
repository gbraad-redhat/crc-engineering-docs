Code signing the macOS installer
================================


## Instructions

This document lists the step I took to codesign the crc installer
  - `make out/macos-universal/crc-macos-installer.tar`
  - copy the tarball to the macOS machine which will sign the installer
  - unpack the tarball
  - set `CODESIGN_IDENTITY` and `PRODUCTSIGN_IDENTITY` to match the certificates you'll be using.

For example:
```bash 
$ export PRODUCTSIGN_IDENTITY="Developer ID Installer: Christophe Fergeau (GSP9DR7D3R)"
$ export CODESIGN_IDENTITY="Developer ID Application: Christophe Fergeau (GSP9DR7D3R)"
```

  - run `packaging/package.sh ./packaging`, this will generate a signed `packaging/crc-macos-installer.pkg` file

This file can now be notarized with 
  - `xcrun notarytool submit --apple-id apple@crc.dev --team-id GSP9DR7D3R --wait ./packaging/crc-macos-installer.pkg`.

> [!NOTE]
> The `--wait` is optional.  
> `xcrun notarytool info` and `xcrun notarytool log` can be used to monitor the progress.

Once the notarization reports `Accepted`, you can run:
  - `xcrun stapler staple ./packaging/crc-macos-installer.pkg` to attach the result to the installer

Afterwards, `spctl --assess -vv --type install ./packaging/crc-macos-installer.pkg` can be used to check the signature and notarization of the .pkg file.
