---
title: Long-term open issues:build determinism
permalink: /Long-term_open_issues:build_determinism/
---

What are the builder-accessible pieces of information that can lead to repeated build producing bit-for-bit different outputs?

### System time

Additional problem: ctime on files, if build creates any archives

#### Ways to fix the problem

##### LD_PRELOAD

Works only for dynamically-linked programs

##### ptrace

Slower?

### System CPU model/features

### Order of directory entries

ls -u giving different order may lead to things like jars being not bit-for-bit identical

### Deliberate randomness

Example: package generating a andom seed during build. It is a problem from many points of view.

Allegedely fixed (at least in Nix on NixOS)
-------------------------------------------

### Builder PIDs