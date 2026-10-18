# sleuthkit

Container images with [The Sleuth Kit](https://www.sleuthkit.org/sleuthkit/), the command-line tools for forensic analysis of disk images. They read DOS, GPT, Mac, BSD and Sun partition tables and NTFS, FAT, exFAT, ext2/3/4, HFS+, APFS, UFS, ISO 9660 and YAFFS2 file systems, list and extract files (deleted ones included), recover files and build timelines. The Sleuth Kit is compiled from the signed release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when it publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by The Sleuth Kit project. Report problems with the image in this repository and bugs in The Sleuth Kit [upstream](https://github.com/sleuthkit/sleuthkit/issues).

## Quick start

Show the partition table of a disk image in the current directory:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" --entrypoint mmls \
  ghcr.io/randomcontainers/sleuthkit disk.img
```

The entrypoint is `fls`, which lists the files of a file system. Pass the start sector of a partition from the `mmls` output with `-o`; `-r` lists subdirectories too, and deleted entries are marked with `*`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/sleuthkit -r -o 2048 disk.img
```

`icat` writes the content of a file to standard output, given the inode number that `fls` prints:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" --entrypoint icat \
  ghcr.io/randomcontainers/sleuthkit -o 2048 disk.img 1234 > file.bin
```

Recover the deleted files of that partition into `recovered/` (`-e` recovers all files, `-a` only allocated ones):

```sh
mkdir -p recovered
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/evidence:ro" -v "$PWD/recovered:/work" \
  --entrypoint tsk_recover ghcr.io/randomcontainers/sleuthkit -o 2048 /evidence/disk.img .
```

Write a timeline of every file's modification, access, change and creation times as CSV, with ISO 8601 dates in UTC:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" --entrypoint sh \
  ghcr.io/randomcontainers/sleuthkit \
  -c 'fls -r -m / -o 2048 disk.img | mactime -d -y' > timeline.csv
```

Each tool prints its options when run without arguments, except `srch_strings`, which then reads standard input; `srch_strings -h` prints its options. The [Sleuth Kit wiki](https://github.com/sleuthkit/sleuthkit/wiki) describes the tools in more detail.

## Working with disk images

- The tools only read images. Mount the evidence with `:ro` as well, and write output to a separate mount, as in the `tsk_recover` example.
- `tsk_loaddb` writes its SQLite database to `<image>.db` next to the image unless `-d` names another file. With the evidence mounted read-only, pass a path on a writable mount, for example `-d /work/case.db` with the evidence in `/evidence`.
- Pass the first file of a split image: `disk.E01` for an EnCase set (the tools find `disk.E02` and the rest in the same directory), `disk.001`, `disk.000` or `disk.aa` for split raw images.
- FAT stores local time. The tools convert it with the container's time zone, which is UTC. For a disk from a machine in another time zone, pass it with `-e TZ=Europe/Berlin` so that `fls -m` and `istat` give the right times. `mactime -z Europe/Berlin` prints a timeline in that zone; `mactime -z list` needs the Perl module DateTime::TimeZone, which the image does not have.
- The image cannot decrypt encrypted APFS or BitLocker volumes: The Sleuth Kit's configure script does not turn on the OpenSSL and Mbed TLS code they need.

The image formats depend on the distro, because Alpine has no packages for most of the libraries The Sleuth Kit uses for them:

| Format | Ubuntu | Alpine |
|---|---|---|
| Raw and split raw (`dd`) | yes | yes |
| EnCase E01 (libewf) | yes | yes |
| AFF (afflib) | yes | no |
| VMDK (libvmdk) | yes | no |
| VHD (libvhdi) | yes | no |
| Linux LVM volume groups (libvslvm, `-P lvm`) | yes | no |

`img_stat -i list` prints the image formats the image supports.

## What is in the image

- In `/usr/local/bin`: `blkcalc`, `blkcat`, `blkls`, `blkstat`, `fcat`, `ffind`, `fiwalk`, `fls`, `fsstat`, `hfind`, `icat`, `ifind`, `ils`, `img_cat`, `img_stat`, `istat`, `jcat`, `jls`, `mactime`, `mmcat`, `mmls`, `mmstat`, `pstat`, `sigfind`, `sorter`, `srch_strings`, `tsk_comparedir`, `tsk_gettimes`, `tsk_imageinfo`, `tsk_loaddb`, `tsk_recover` and `usnjls`.
- The Sleuth Kit's library, `libtsk`, in `/usr/local/lib`, and the category files of `sorter` in `/usr/local/share/tsk/sorter`.
- The distro's SQLite (for `tsk_loaddb` and hash databases) and zlib. On Ubuntu, the distro's libewf, afflib, libvhdi, libvmdk and libvslvm. On Alpine, libewf 20140817 from the [libewf-legacy](https://github.com/libyal/libewf-legacy) series in `/usr/local/lib`, the series Ubuntu's libewf comes from. Alpine's own libewf package is the newer experimental series, whose API The Sleuth Kit does not support.
- Perl for `mactime` and `sorter`, `file` for `sorter`, and the time zone data.

Not included: the Java bindings, the headers, the pkg-config file and the `libtsk.so` link for building against `libtsk`, the man pages, and `jpeg_extract`, a `fiwalk` plugin that runs the `exif` program, which the image does not have. The configure summary, `fiwalk -V` and the list of image formats are in `/usr/local/share/randomcontainers/sleuthkit/buildinfo`.

## Default or slim

The Sleuth Kit's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a release of The Sleuth Kit such as `4.15.0`. `<minor>` and `<major>` are its shorter forms, `4.15` and `4`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current version of The Sleuth Kit, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

The user in the container needs read access to the image files. `tsk_recover` writes the recovered files with the UID and GID the container runs as, not their owners on the original disk.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new release of The Sleuth Kit and are rebuilt when the base image changes. The packages The Sleuth Kit needs are listed in `/usr/local/share/randomcontainers/sleuthkit/runtime-deps`. Switch to root to install more, then back. This adds ExifTool, to read the metadata of recovered files:

```dockerfile
FROM ghcr.io/randomcontainers/sleuthkit:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends libimage-exiftool-perl \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache exiftool`. The entrypoint is `["tini", "--", "fls"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new releases of The Sleuth Kit and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/sleuthkit:latest \
  --repo randomcontainers/sleuthkit --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/sleuthkit:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against Brian Carrier's release signing key in `keys/sleuthkit-release.gpg`, fingerprint `0917 A7EE 58A9 308B 13D3 9633 38AD 602E C745 4C8B`. That key is a DSA-1024 key from 2004 and signs with SHA-1, which no longer count as strong, so the SHA-256, taken from the digest GitHub records for the release asset, is the main check and the signature a second one. On Alpine, the build checks the libewf tarball against its SHA-256 in `package.yml`, which is the digest GitHub records for it, and its signature against Joachim Metz's key in `keys/libewf-release.gpg`, fingerprint `0ED9 020D A90D 3F6E 70BD 3945 D962 5E5D 7AD0 177E`. On Ubuntu, the build adds `tsk/pool/tsk_lvm.hpp`, which the LVM support needs and the release tarballs leave out, from the release tag named in `package.yml`, and checks it against its SHA-256 there.

## Updates

The project checks the releases of [sleuthkit/sleuthkit](https://github.com/sleuthkit/sleuthkit/releases) every 15 minutes and skips pre-releases. A release is picked up once it is 24 hours old. Its tarball is checked against the SHA-256 digest GitHub recorded for the release asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. A release signed with a key that is not in `keys/sleuthkit-release.gpg` fails to build until the key is added. Only the newest release is built; tags of older versions stay as they were last built.

libewf on Alpine is pinned to a version in `package.yml` and does not follow the releases of libewf-legacy automatically. So is `tsk_lvm.hpp` on Ubuntu, which has not changed since 2022. A release whose tarball has its own copy of that file fails to build if the copy differs.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  --build-arg TSK_LVM_HEADER_VERSION=<version> \
  --build-arg TSK_LVM_HEADER_SHA256=<sha256> \
  -t sleuthkit:local .
```

`TSK_LVM_HEADER_VERSION` and `TSK_LVM_HEADER_SHA256` are the `version` and `sha256` of the `tsk-lvm-header` entry under `extra-artifacts` in `package.yml`. For the Alpine image, use `Dockerfile.alpine`, leave those two out and pass `--build-arg LIBEWF_VERSION=<version>` and `--build-arg LIBEWF_SHA256=<sha256>`, the `version` and `sha256` of the `libewf` entry under `extra-artifacts` in `package.yml`. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

The Sleuth Kit is licensed under the Common Public License 1.0 (CPL-1.0). The code it took over from The Coroner's Toolkit, and the HFS+ and ISO 9660 code contributed by Crucial Security and ATC-NY, are licensed under the IBM Public License 1.0 (IPL-1.0). Code compiled in under other licenses:

- GPL-2.0-or-later: `srch_strings`, a separate program taken from GNU binutils (`GNUv2-COPYING`).
- BSD-3-Clause: the LZVN decoder from Apple and the SHA-2 code in `fiwalk` (`COPYING.lzvn`, `COPYING.sha2`).
- MIT: the GUID code in `libtsk` (`COPYING.guid`).
- ISC, with a permissive notice from IBM: the base64 code in `fiwalk` (`COPYING.base64`).
- RSA-MD: the MD5 code, derived from the RSA Data Security, Inc. MD5 Message-Digest Algorithm (`COPYING.md5c`).
- The Unicode ConvertUTF notice: the UTF-8 and UTF-16 conversion code (`COPYING.tsk_unicode`).
- Dan Farmer's notice: `mactime`, which is based on his original program (`COPYING.mactime`).
- Public domain: the CRC and SHA-1 code, the zlib example code in the decompression routines, and most of `fiwalk`.

The files named above are in `/usr/local/share/randomcontainers/sleuthkit/licenses/`, with the license texts from the tarball (`cpl1.0.txt`, `IBM-LICENSE`, `bsd.txt` and `mit.txt`) and its `README.md`, which lists the copyright notices. On Alpine the directory also has `libewf/`, with libewf's `AUTHORS`, `COPYING` and `COPYING.LESSER`: libewf is licensed under the GNU Lesser General Public License version 3 or later (LGPL-3.0-or-later).

The image's license label is `CPL-1.0 AND IPL-1.0 AND GPL-2.0-or-later AND LGPL-3.0-or-later AND BSD-3-Clause AND MIT AND ISC AND RSA-MD AND LicenseRef-IBM-base64 AND LicenseRef-ConvertUTF AND LicenseRef-mactime AND LicenseRef-Public-Domain`, where the `LicenseRef-` entries stand for the IBM, ConvertUTF, Dan Farmer and public domain notices.

The Ubuntu and Alpine packages in the image keep their own licenses. Ubuntu's afflib, which reads AFF images, is under the 4-clause BSD license, which requires this acknowledgement: "This product includes software developed by Simson L. Garfinkel and Basis Technology Corp."

The corresponding source for each image:

- The Sleuth Kit: every version has a GitHub release in this repository, named `v<version>`, with the exact `sleuthkit-<version>.tar.gz` that was compiled and its signature, the `libewf-<version>.tar.gz` compiled on Alpine with its signature, and the `tsk_lvm.hpp` that the Ubuntu build adds. The build applies no patches. `/usr/local/share/randomcontainers/sleuthkit/source` lists that release and the upstream download URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
