Building the .msi package
=========================


## Requirements
  1. Windows box with `make` and `git` installed. (the msi can only be built on a windows machine since it uses [wixtoolset](https://wixtoolset.org). not tested on other platforms!)
  2. powerhshell core is installed (Makefile targets for msi makes use of `pwsh`)
  3. [WixTool Set](https://wixtoolset.org/releases/) is installed
  4. The Hyper-V bundle that needs to be in the msi

## How to build
Following commands are tested in `PowerShell`, `PowerShell Core` and `Cygwin`
  - To build the msi run: `make BUNDLE_DIR=<path_to_dir_containing_hyperv_bundle> out/windows-amd64/crc-windows-amd64.msi`
  - To build the zip archive (release artifact) run: `make BUNDLE_DIR=<path_to_dir_containing_hyperv_bundle> out/windows-amd64/crc-windows-installer.zip`

For testing purposes the msi can be built without including the bundle using `make MOCK_BUNDLE=true BUNDLE_DIR=./ out/windows-amd64/crc-windows-amd64.msi`

## Build process
The msi is built using the commands [`candle.exe`](https://wixtoolset.org/documentation/manual/v3/overview/candle.html) and [`light.exe`](https://wixtoolset.org/documentation/manual/v3/overview/light.html) tools provided by the wixtoolset.

The [product.wxs.in](https://github.com/crc-org/crc/blob/master/packaging/windows/product.wxs.in) file is the main wxs file that describe the msi product, among other things it mentions the files that needs to be put in _C:\Program Files\Red Hat OpenShift Local_, the [LicenseAgreementDlg_HK.wxs](https://github.com/code-ready/crc/blob/master/packaging/windows/LicenseAgreementDlg_HK.wxs) and [WixUI_HK.wxs](https://github.com/crc-org/crc/blob/master/packaging/windows/WixUI_HK.wxs) files respectively describe the look of the license agreement dialog and the overall installer flow.

> [!NOTE]
> These boilerplate files were first obtained by using [mh-cbon/go-msi](https://github.com/mh-cbon/go-msi) and modifying them to our need.

We include `crc-admin-helper` in the msi, these files are converted to cab archives during the msi build process by `light.exe`.

> [!NOTE]
>  there's a limit to how big a cab file can be, therefore we split the hyperv bundle into chunks of 1GB using the code at [split.go](https://github.com/code-ready/crc/blob/master/packaging/windows/split.go))

Finally after the msi is built, we package the cab files and msi together in a zip file, the Makefile target `out/windows-amd64/crc-installer-windows.zip` generates the final releasable artifact which can be found in the folder _`out/windows-amd64/`_.

### Resources
- WixTool set official tutorial (https://www.firegiant.com/wix/tutorial)
- Examples of wxs files on github
- Wix Custom Action (Quitely execute) https://wixtoolset.org/documentation/manual/v3/customactions/qtexec.html
- Enable Windows Optional Feature https://stackoverflow.com/questions/18126502/wix-enable-windows-feature