Releasing on GitHub
===================


## Releasing using the github actions workflow

The GitHub Actions workflow [`Publish release on github`](https://github.com/crc-org/crc/actions/workflows/release.yml) creates a draft release and provides a template with all the component versions and the git change log.

To start the workflow go to the [workflow page](https://github.com/crc-org/crc/actions/workflows/release.yml) and click on the `Run Workflow` button, make sure to choose the appropriate tag for the release.

Once the draft release is available, edit it to include the notable changes for the release and press publish to make it public.


## Releasing using the `gh-release.sh` script

In the CRC repository, we have a script [`gh-release.sh`](https://github.com/crc-org/crc/blob/main/gh-release.sh) which uses the [`gh`](https://cli.github.com) tool, make sure it is installed.

Create a markdown file containing a list of the notable changes named `notable_changes.txt` in the same directory as the script.

An example `notable_changes.txt`:
```bash
$ cat notable_changes.txt
- Fixes a bug where `oc` binary was not extracted from bundle when using microshift preset [#3581](https://github.com/crc-org/crc/issues/3581)
- Adds 'microshift' as a possible value to the help string of the 'preset' config option [#3576](https://github.com/crc-org/crc/issues/3576)
```

Then run the script from the release tag and follow the prompts, it’ll ask for confirmation before pushing the draft release to GitHub.

> [!NOTE]
> The script will exit with error if it doesn’t find a tag starting with `v` in the current git HEAD.

```bash
$ git checkout v2.18.0
$ ./gh-release.sh
```

Verify the draft release on the releases page and if everything looks good press publish to make the release public.
