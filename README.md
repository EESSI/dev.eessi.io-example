# `dev.eessi.io` Example Project

This is an example repository for the setup of the build and ingestion process for EESSI's `dev.eessi.io` CernVM-FS repository. This project is currently primarily intended for the [MultiXscale CoE](https://multixscale.eu) partners working on the development of the project's core applications.

`dev.eessi.io` project repositories are hosted on GitHub and follow the naming convention: `dev.eessi.io-projectname`. Once built and ingested, installations for each project will be available in systems that have EESSI available under `/cvmfs/dev.eessi.io/projectname/2023.06`, where `2023.06` corresponds to the matching version of EESSI's `software.eessi.io` repository and [compatibility layer](https://github.com/EESSI/filesystem-layer).

## Installation instructions for new project repositories

This section details the steps needed to setup the infrastructure for a new project's repository and bot.

Setting up an instance of the bot for a dev.eessi.io project mostly consists of following the instructions detailed at the bot's [development repository](https://github.com/EESSI/eessi-bot-software-layer). There are however a few details and differences:

### app.cfg
The bot's `app.cfg` file needs to be adjusted to the cluster it is running on. Besides the obvious (app ID that points to the correct GitHub App, private key specific to this bot, etc,), these are:
``` ini
[buildenv]
build_job_script = bot/bot-build-dev.eessi.io.slurm@git@github.com:EESSI/dev.eessi.io-scripts.git
```
(This also requires an additional key pair, see details later!)

``` ini
[deploycfg]
...
bucket_name = { "dev.eessi.io": "dev.eessi.io-2024.09" }
```
If you are deploying the tarballs to an AWS S3 bucket, keep in mind you will need to install the necessary tooling and set it up following the instructions in the bot's [documentation](https://github.com/EESSI/eessi-bot-software-layer?tab=readme-ov-file#step-41-installing-tools-to-access-s3-bucket). You will also need the associated key pair for the bucket in question.


For the currently available architectures in the dev.eessi.io cluster:
``` ini
[repo_targets]
# defines for which repository a arch_target should be built for
repo_target_map = {
    "linux/x86_64/amd/zen2" : ["dev.eessi.io"],
    "linux/x86_64/amd/zen4" : ["dev.eessi.io"],
    "linux/aarch64/neoverse_n1" : ["dev.eessi.io"] }
    "linux/aarch64/neoverse_v1" : ["dev.eessi.io"] }
```

### repos.cfg
The `repos.cfg` file needs to be configured for `dev.eessi.io`. All that needs to be changed is a section named `[dev.eessi.io]` with `repo_name=dev.eessi.io`. All other settings can remain the same as the production bot. You can find more detailed information about `repos.cfg` in the relevant section of the bot's [documentation](https://github.com/EESSI/eessi-bot-software-layer/blob/develop/README.md#repo_targets-section).

### Build script key pair
The `dev.eessi.io` bot uses a different script to start build jobs. This script is at https://github.com/EESSI/dev.eessi.io-scripts and is obtained through git from that repository (the repository and the script can be changed via the `[buildenv]` section in `app.cfg`). To do so, a key pair with at least read access is required in the GitHub account that owns the bot instance (most likely this will be https://github.com/EESSIbot). If setting up a new instance, a new pubkey should be [given to that GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) (currently can be done by @boegel and @bedroge), while the corresponding private key should be in `$HOME/.ssh` of the `bot` account in the build cluster.

### Sign key
To deploy builds, tarballs now need to be signed and verified. Sign the tarballs, make sure you fill in the `signing` item under the `[deploycfg]` section of `app.cfg`:
``` ini
# settings for signing artefacts with JSON-like format
#   REPO_ID: { "script": PATH_TO_SIGN_SCRIPT, "key": PATH_TO_KEY_FILE, "container_runtime": PATH_TO_CONTAINER_RUNTIME }
#  If PATH_TO_SIGN_SCRIPT is a relative path, the script must reside in the
#  checked out pull request of the target repository (e.g.,
#  EESSI/software-layer).
#  The bot calls the script with the two arguments:
#   1. private key (as provided by the attribute 'key')
#   2. path to the file to be signed (the upload script will determine that)
# NOTE (on "container_runtime"), signing requires a recent installation of OpenSSH
#   (8.2 or newer). If the frontend where the event handler runs does not have that
#   version installed, you can specify a container runtime via the 'container_runtime'
#   attribute below. Currently, only Singularity or Apptainer are supported.
# NOTE (on the key), make sure the file permissions are restricted to `0600` (only
#   readable+writable by the file owner, or the signing will likely fail.
# Note (on json format), make sure no trailing commas are used after any elements
#   or parsing/loading the json will likely fail. Also, the whole value should start
#   at a new line and be indented as shown below.
signing =
    {
        "dev.eessi.io": {
            "script": "/path/to/cloned/bot/repository/eessi-bot-software-layer/scripts/sign_verify_file_ssh.sh",
            "key": "/path/to/key/a_key_file.pem",
            "container_runtime": "/usr/bin/singularity"
        }
    }

```

Make sure to also note down what you chose for `app_name` at the top of the file. You will need this (and it has to match exactly), the pubkey, and the date the key was created. This information has to be added through a PR on a separate (private) repository so that the tarball signing can be verified before ingestion. Without this, deployed tarballs will not open staging PRs and can't be ingested.

