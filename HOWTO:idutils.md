---
title: HOWTO:idutils
permalink: /HOWTO:idutils/
---

    # create database:
    mkid

    # use the database maybe this way
    lid -R grep -r 'your regex'

    # too much to type? put into your .bashrc then use: mylid 'your regex'
    mylid(){ lid -R grep -r "$1"; }

This is useful to find options in nixos (eg grep for mkOption) or find packages in nixpkgs. You may also use this to find names in source code (-&gt; see note). For code navigation there may be more accurate tools such as ctags or csope etc.

note: you may have to use a custom config. idutils only index files it knows about. Nixpkgs contains a patch which adds .nix files to the global config file. AFAIK README files etc are skipped by default. So if you don't find what you're looking for you still may want to use plain grep.

If you want to search for "foo bar" you can still use a pipe:

    lid -R grep -r 'foo' | grep 'foo bar'

[Category:Installation](/Category:Installation "wikilink") [Category:Configuration](/Category:Configuration "wikilink")