# kubecp
Kube Copy (kubecp) copies directory contents from and to kubernetes pods while preserving the permissions and ownership of the files.

## Overview

kubecp can be used to:
- migrate data from one pod to another (even across different namespaces or clusters) - **default behavior**
- backup data from a pod to your local machine - (see `--download-only`)
- upload or restore data to a pod from your local machine - (see `--upload-only`)

While this seems fairly simple, the vast majority of the script is designed to provide:
- options to modify default behavior
- safe-guards to prevent unintended consequences
- verification that the provided contexts, namespaces, pods, containers, & directories actually exist
- feedback during long running processes

By default, the script runs in `interactive` mode and prompts the user for any missing values or potentially destructive operations.

## Requirements

- The machine where this is run needs the following:
  - `kubectl`: installed and configured properly
  - `file`: BSD cli utility
  - `awk`
- Source and destination containers need the following:
  - `tar`
  - `ls`
  - `rm`

## Basic Installation

For Linux & Mac:

```bash
sudo wget https://raw.githubusercontent.com/dalehenries/kubecp/main/kubecp
sudo chmod +x kubecp
sudo mv kubecp /usr/local/bin/.
```

## Preserving File Ownership & Permissions

For the permissions and ownership to be preserved, the user on the pods needs sufficient permissions to read/write to the chosen directories.

Why not just use `kubectl cp`?  `kubectl cp` also uses `tar` to download the files to your local machine, but it also extracts the files there.  Since most people don't run their `kubectl` commands with elevated permissions, the ownership of the extracted files is changed to your current user if your current user doesn't have permissions to write files as the original owner.  When the files are only extracted on the destination pod, as long as that user has permission to write the files as the original owner from the source pod, ownership is preserved.


## Options

| NAME                         |  TYPE  | DESCRIPTION                                                      | DEFAULT                                                      |
| :--------------------------- | :----: | :--------------------------------------------------------------- | :----------------------------------------------------------- |
| `-h`,<br />`--help`          |        | Print usage info and exit                                        |                                                              |
| `--version`                  |        | Print version and exit                                           |                                                              |
| `-i`,<br />`--interactive`   |  bool  | Prompt user for missing information                              | `true`                                                       |
| `-p`,<br />`--src-pod`       | string | Name of pod to copy files from                                   |                                                              |
| `-d`,<br />`--src-dir`       | string | Path to directory to copy files from                             |                                                              |
| `-x`,<br />`--src-context`   | string | Kubectl context where `--src-pod` is                             | current `kubectl` context                                    |
| `-n`,<br />`--src-namespace` | string | Namespace `--src-pod` is  in                                     | current namespace of `--src-context` or `default`            |
| `-c`,<br />`--src-container` | string | Name of container in `--src-pod` to copy<br /> files from        |                                                              |
| `-P`,<br />`--dst-pod`       | string | Name of pod to copy files to                                     |                                                              |
| `-D`,<br />`--dst-dir`       | string | Path to directory to copy files. to                              |                                                              |
| `-X`,<br />`--dst-context`   | string | Kubectl context where `--dst-pod` is                             | current `kubectl` context                                    |
| `-N`,<br />`--dst-namespace` | string | Namespace `--dst-pod` is  in                                     | current namespace of `--dst-context` or `default`            |
| `-C`,<br />`--dst-container` | string | Name of container in `--dst-pod` to copy<br /> files to          |                                                              |
| `--overwrite-dst`            |  bool  | Contents of `--dst-dir` will be<br /> overwritten when not empty | `false`                                                      |
| `--download-only`            |  bool  | Skip upload, only download files to<br /> local machine          | `false`                                                      |
| `--upload-only`              |  bool  | Skip download, only upload a gzipped<br /> archive to --dst-pod  | `false`                                                      |
| `--keep-local`               |  bool  | Leave copy of downloaded archive on<br /> local machine          | `false`,<br />`true` IF `--download-only` OR `--upload-only` |
| `--dry-run`                  |  bool  | Do everything but don't actually<br /> download/upload anything  | `false`                                                      |
| `--local-file`               | string | Name of the gzipped archive stored on<br /> local disk           | `archive.tar.gz`                                             |

Options can be pass with either `--option=value` or `--option value`.

Boolean options can be passed without a value and are then considered `true`.

## Interactive Mode

The default behavior is to run in interactive mode. In interactive mode, the user will be prompted to provide any additional information needed at runtime.

For use in automation, interactive mode can be disabled with `--interactive false`.

When interactive mode is disabled, some of the options may become required. We recommend running with `--dry-run` before automation to verify all required options are passed.

## Examples

```bash
# copy the static html files from one nginx pod to another running on the same cluster in the same namespace
kubecp --src-pod=old-nginx --src-dir=/usr/share/nginx/html --dst-pod=new-nginx --dst-dir=/usr/share/nginx/html

# download static website files from an nginx container
kubecp --src-context developer@cluster1 \
  --src-namespace project-one \
  --src-pod my-site-nginx \
  --src-dir /usr/share/nginx/html \
  --download-only \
  --local-file my-site-nginx-backup.tar.gz

# copy wordpress uploads from a pod on one cluster to a pod in a different namespace on a different cluster
# overwrite contents of upload directory even if it isn't empty
# keep a copy of the archive on the local machine
kubecp --src-context developer@cluster1 \
  --src-namespace project-one \
  --src-pod my-wordpress-site \
  --src-dir /var/www/html/wp-content/uploads
  --dst-context developer@cluster2 \
  --dst-namespace project-two \
  --dst-pod my-other-wordpress-site \
  --dst-dir /var/www/html/wp-content/uploads
  --overwrite-dst
  --keep-local

```

## Basic Procedure Overview

The default behavior does the following:
1. Downloads a specified directory from the source pod to the local machine as a gzipped archive (`.tar.gz`)
2. Uploads the archive to the destination pod
3. Extracts the contents on the destination pod
4. Deletes the archive from the destination pod and the local machine

The commands used to do these operations are as follows:

```bash
# Download files from --src-pod to local machine
kubectl --context [src-context] -n [src-namespace] exec -i [src-pod] -c [src-container] -- tar czpf - --directory=[src-dir] . > archive.tar.gz

# Upload archive.tar.gz to --dst-pod
kubectl --context [dst-context] -n [dst-namespace] cp archive.tar.gz [dst-pod]:/tmp/ -c [dst-container]

# Extract contents of archive.tar.gz on --dst-pod at --dst-dir
kubectl --context [dst-context] -n [dst-namespace] exec -i [dst-pod] -c [dst-container] -- tar xzpf /tmp/archive.tar.gz --directory=[dst-dir]

### Delete archive.tar.gz from --dst-pod
kubectl --context [dst-context] -n [dst-namespace] exec -i [dst-pod] -c [dst-container] -- rm /tmp/archive.tar.gz

### Delete archive.tar.gz from local machine
rm archive.tar.gz
```
