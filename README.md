# MOS Sundtek Driver

mos-sundtek-driver provides a **MOS plugin** for the
[Sundtek DVB Driver](https://sundtek.com)
that integrates with TVHeadend Docker containers.

---

## Overview

This repository contains the MOS plugin implementation. The Sundtek
installation script is downloaded from sundtek.de when the plugin is
first installed or when the driver is updated.

The plugin allows MOS to install and manage the Sundtek DVB driver
within TVHeadend Docker containers.

### Script Source

- Sundtek Installation Script: [https://sundtek.de](https://sundtek.com)

---

## Build & Automation

This repository includes a **GitHub Actions workflow** used to build and
package the plugin for MOS.

The build process is fully automated and produces artifacts that can be
installed through the MOS Hub.

---

## Licensing

The contents of this repository (plugin code, scripts, configuration,
and automation) are licensed under **GPL-3.0**.

The Sundtek installation script remains licensed under its respective
upstream license.
