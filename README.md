# Nemo Actions for Audio Library Maintenance

Linux Nemo file manager right-click actions for verifying, tagging, and applying ReplayGain to an audio library from within Nemo. Companion to the SHA-512 checksum and moOde cleanup guides.

**Guide version: v1** — Merges the SHA512 and ReplayGain actions into a single guide, and adds tag-verification and tag-writing actions.

## Overview

This repository contains a set of Nemo right-click actions:

* **Verify ALBUM SHA512 Checksums** — checks individual track files against an `ALBUM.sha512sums.txt` manifest.
* **Verify ARTIST SHA512 Checksums** — computes a hash-of-hashes across album directories inside an artist folder and compares them against `ARTIST.sha512sums.txt`.
* **Apply ReplayGain** — applies ReplayGain to a selected album or artist.
* **Report Tag/Filename Mismatches** — compares embedded tags against the folder/filename naming convention.
* **Write Tags from Folder/File Names** — losslessly writes Artist/Album/Year/Title/TrackNumber tags from the naming convention.

## Prerequisites

Ensure you have the following on your system:

* Linux Mint with the Nemo file manager
* Terminal access
* `sha512sum`, `flac`, `metaflac`, `ffmpeg`/`ffprobe`, `eyeD3`, `AtomicParsley`, and `loudgain` (the guide lists these per action)

## Recommended Workflow

A four part series to clean, verify, and lockdown securely the integrity of an audio file library.

1. linux-audio-moode-prep: https://github.com/TerrapinATL/linux-audio-moode-prep

2. linux-audio-sha512-checksums: https://github.com/TerrapinATL/linux-audio-sha512-checksums

3. linux-os-nemo-sha512-shortcut: https://github.com/TerrapinATL/linux-os-nemo-sha512-shortcut

4. linux-audio-folder-recertification: https://github.com/TerrapinATL/linux-audio-folder-recertification

## Disclaimer

This file was created as a mix of AI generated content, user input, and user editing. It was a cooperative effort between Claude, Gemini, ChatGPT, and user.

## IMPORTANT

Your Original Library should be treated as immutable.

You should only work on a COPY of your Original Library when processing these scripts. The workflow is designed around creating a validated secondary copy, testing the results, and only then promoting that copy to become a replacement.

Before promotion, files should be cleaned, verified with `flac -t`, and protected with two layers of SHA-512 checksums.

The purpose is to ensure you have a verifiable library that can be copied, backed up, and restored repeatedly while still matching the validated cleaned copy.