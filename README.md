# XRP_Firmware

This repository contains XRP firmware and XRPLib for use with the XRP IDE. The firmware and library are in the boards folder. The folders are named based on the XRP board type. For example, the beta XRP is in the `xrp-beta` folder, and the production XRP is in the `xrp-2350` folder.

## Release Process

### Adding a new XRP type to the IDE

To add a new board to the IDE, you will need to create a new folder in the boards folder and add the firmware and library to it.

### Updating the release firmware and library

To update a new board firmware release and XRP library, you will need to create a new tag and push it to the repository. Then the GitHub Actions workflow will automatically create a release with the same name as the tag. See the release section for more information.

### Create a new release

1. **Create a new tag**

   ```bash
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```

   where X is the major version, Y is the minor version, and Z is the patch version. For example, if the current version is v1.0.0, the next version will be v1.0.1.

2. **The GitHub Actions workflow will automatically create a release**
   The workflow `release.yml` will trigger on the new tag and create a release with the same name as the tag.
