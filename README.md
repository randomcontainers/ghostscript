# ghostscript

Container images with [Ghostscript](https://www.ghostscript.com/), the PostScript and PDF interpreter, for converting, rendering, merging and compressing documents from the command line. Ghostscript is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when Artifex publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by Artifex Software, which develops Ghostscript. Report problems with the image in this repository and Ghostscript bugs at [bugs.ghostscript.com](https://bugs.ghostscript.com/).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" randomcontainers.com/ghostscript \
  -sDEVICE=pdfwrite -o output.pdf input.ps
```

The same images are published as `ghcr.io/randomcontainers/ghostscript`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ghostscript \
  -sDEVICE=png16m -r150 -o page-%03d.png input.pdf
```

`randomcontainers.com/ghostscript` is a pull-only alias for the GitHub Container Registry images, meant for typing at a shell. Use the `ghcr.io` name in CI, Kubernetes and `FROM` lines, pinned by digest (`ghcr.io/randomcontainers/ghostscript@sha256:...`). Pulling a tag through the alias means trusting the project's Cloudflare account to return the right image. A pull by digest does not depend on that, because Docker checks the content against the digest, and the attestation check under [Verifying](#verifying) reads from ghcr.io directly.

The entrypoint runs `gs` under `tini` in `/work`, so file names are relative to the directory you mount. A few more commands:

```sh
# Make a PDF smaller by downsampling its images
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" randomcontainers.com/ghostscript \
  -sDEVICE=pdfwrite -dPDFSETTINGS=/ebook -o smaller.pdf input.pdf

# Merge PDFs
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" randomcontainers.com/ghostscript \
  -sDEVICE=pdfwrite -o merged.pdf first.pdf second.pdf

# Run one of the helper scripts, such as ps2pdf, pdf2ps or eps2eps
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint ps2pdf \
  randomcontainers.com/ghostscript input.ps output.pdf
```

The [Ghostscript documentation](https://ghostscript.readthedocs.io/en/latest/Use.html) covers the options and output devices.

## What is in the image

- `gs` and its helper scripts in `/usr/local/bin`. `dvipdf` is left out because it needs `dvips` from TeX.
- The PostScript resources and the URW base 35 fonts in `/usr/local/share/ghostscript/<version>/`. Times, Helvetica, Courier and the other standard PostScript fonts resolve to these.
- The distro's FreeType, fontconfig, libjpeg-turbo, libpng, LibTIFF, OpenJPEG, jbig2dec, Brotli and zlib. Artifex's thread-safe fork of Little CMS (lcms2mt) is compiled in.

Not included: CUPS output, the X11 and GTK display devices, the contributed printer drivers from the tarball's `contrib/` directory, libpaper, libidn (used for Unicode PDF passwords) and the OCR devices, which need Tesseract. The default page size is US Letter; pass `-sPAPERSIZE=a4` for A4 when a PostScript file does not set its own size. The configure flags are in `/usr/local/share/randomcontainers/ghostscript/buildinfo`.

## Default or slim

Every randomcontainers package has a `latest` image for general use and a `slim` image to build on. Ghostscript's default image adds no other tools, so here both tags point to the same image. For ImageMagick with PDF and PostScript support, use the [imagemagick](https://randomcontainers.com/imagemagick/) image, whose default includes this build of Ghostscript.

## Tags

`<version>` is a Ghostscript release such as `10.08.0`. `<minor>` and `<major>` are its shorter forms, `10.08` and `10`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

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

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Ghostscript version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time. The [image page](https://randomcontainers.com/ghostscript/) lists the current tags.

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

Ghostscript writes temporary files to `/tmp`, so a container started with `--read-only` also needs `--tmpfs /tmp`.

## Untrusted files

PostScript is a programming language, and Ghostscript has had several sandbox escapes, for example CVE-2023-36664 and CVE-2024-29510. Ghostscript runs with its sandbox (`-dSAFER`) on by default, and these images keep that default. Do not pass `-dNOSAFER` when processing files you did not create. For files from unknown sources, also take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --network none --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --pids-limit 64 \
  randomcontainers.com/ghostscript -sDEVICE=pdfwrite -o clean.pdf untrusted.pdf
```

Mount only the directory the job needs. Ghostscript releases often fix security issues, and the images pick up a new release about a day after Artifex publishes it.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Ghostscript release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/ghostscript:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends fonts-liberation \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

Ghostscript finds fonts installed this way through fontconfig. On Alpine, start from `slim-alpine` and use `apk add --no-cache font-liberation`. The entrypoint is `["tini", "--", "gs"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new Ghostscript releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/ghostscript/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages Ghostscript needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/ghostscript:latest \
  --repo randomcontainers/ghostscript --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/ghostscript:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`.

## Updates

The project checks the releases of [ArtifexSoftware/ghostpdl-downloads](https://github.com/ArtifexSoftware/ghostpdl-downloads/releases) every 15 minutes and skips release candidates. A release is picked up once it is 24 hours old. Its tarball is checked against the release's `SHA512SUMS`, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

Tags of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so they pick up distro security fixes.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t ghostscript:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

Ghostscript is licensed under the GNU Affero General Public License, version 3 or later (AGPL-3.0-or-later). The URW fonts are under the same license, with an exception that allows embedding them in PostScript and PDF documents. Other parts have their own licenses:

- Apache-2.0: the Droid Sans Fallback font, and the `snprintf` and `strtok` code from Apache APR. The license text is in `DroidSansFallback.NOTICE`.
- BSD-3-Clause: the SHA-2 code (`COPYING.sha2`), the AES code from XySSL (`COPYING.aes`) and the Adobe CMap files, which have the license at the top of each file.
- FTL: the TrueType bytecode interpreter, which is based in part on the work of the [FreeType Team](https://freetype.org/) (`FTL.TXT`).
- ISC: `strlcpy` and `strlcat` from OpenBSD (`COPYING.gsstrl`).
- MIT: Little CMS (lcms2mt) and IJS (`COPYING.lcms2mt`, `COPYING.ijs`).
- Zlib: the MD5 code (`COPYING.gsmd5`).
- A permissive notice from Lucent Technologies, which has no SPDX identifier: the `inferno` output device (`COPYING.gdevifno`).

Everything in the list except the font and the CMap files is compiled into `gs`. The files named above are in `/usr/local/share/randomcontainers/ghostscript/licenses/`, with Ghostscript's `LICENSE` and `COPYING`. The image's license label is `AGPL-3.0-or-later AND Apache-2.0 AND BSD-3-Clause AND FTL AND ISC AND MIT AND Zlib`. The contributed printer drivers in the tarball's `contrib/` directory, some of which are under GPL-2.0-or-later, are not built. The Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- Ghostscript: every version has a GitHub release in this repository, named `v<version>`, with the exact `ghostscript-<version>.tar.xz` that was compiled. The build applies no patches. It only deletes the tarball's bundled copies of the libraries it takes from the distro, and the bundled Tesseract, Leptonica and CUPS libraries, which are not built. The download URL is in `/usr/local/share/randomcontainers/ghostscript/source`.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The ImageMagick default image and the combined images that include Ghostscript are built on a slim Ghostscript image from this repository. Their `com.randomcontainers.members` label records the Ghostscript version and the digest of that image, whose own `org.opencontainers.image.revision` label names the commit here.

If you redistribute these images, or offer a modified Ghostscript to users over a network, read what the AGPL requires of you. Artifex also sells [commercial licenses](https://artifex.com/licensing/) for use under other terms. A donation to randomcontainers is not a license from Artifex and does not replace one.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Support

The images cost nothing to use. If they save you time, you can support the project at [randomcontainers.com/donate](https://randomcontainers.com/donate/). Artifex funds Ghostscript development by selling commercial licenses, so if your use needs one, buying it also supports Ghostscript.
