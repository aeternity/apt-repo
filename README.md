README.md

## Usage

An example for official releases on Ubuntu 18.04:

```bash
sudo apt-get install software-properties-common
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys D18ABB45C5AF91A0D835367E2106A773A40AFAEB
sudo apt-add-repository 'deb https://apt.aeternity.io bionic main'
sudo apt-get install aeternity-node
```

## Structure

The supported linux distribution codenames are used as APT repository codename as well.

Components:
- main - "Stable" releases
- testing - Development builds based on the `master` branch

## Automation

Currently this repository is run manually.

Automation todo:
- automatically process "incomming" packages and add them to the repo
- automatically sync with the S3 bucket

## Manually update the repository

First make sure you add the package in the root repository dir.

A couple of docker images will be used to run the commands for consistent results.

Starting with a Ubuntu 18.04 as builder image:

```bash
docker run -it -v ${PWD}:/src -w /src ubuntu:18.04
```

The repository is maintained using the `reprepro` tool and needs to be installed first.

```bash
apt-get update && apt-get install -y reprepro unzip curl
```

GPG private key is required to sign repo metadata, it could be received from Vault manully or by installing Vault:

```bash
VAULT_VERSION=0.11.2
curl -sSO https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_0.11.2_linux_amd64.zip \
    && unzip vault_${VAULT_VERSION}_linux_amd64.zip -d /bin \
    && rm -f vault_${VAULT_VERSION}_linux_amd64.zip
```

Import the GPG private key from Vault:

```bash
VAULT_ADDR=https://..
VAULT_TOKEN=your_token
vault read -field=private secret/gpg/D18ABB45C5AF91A0D835367E2106A773A40AFAEB | gpg --import
```

Add new package to the `main` component of bionic distribution:

```bash
reprepro -b repo/ -C main includedeb bionic aeternity-node*.deb
```

And cleanup:
```bash
rm *.deb
```

Exit the builder container and start a new one with the infrastructure console to sync with S3 bucket:

```bash
docker run -it -e AE_VAULT_GITHUB_TOKEN -e AE_VAULT_ADDR -v ${PWD}:/src aeternity/infrastructure make envshell
```

Sync with S3 bucket:
```bash
aws s3 sync --acl public-read --delete /src/repo/ s3://aeternity-node-apt
```

Finnally quit the infra continer and commit & push to the `master` branch the new repository contents:
```bash
git add -A && git commit -m "Update APT repo" && git push
```
