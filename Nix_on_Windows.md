---
title: Nix on Windows
permalink: /Nix_on_Windows/
---

viric's findings
----------------

(viric) I think Side-by-side components are the way to go, so DLL are not taken as \*strictly shared\* by Windows only based on the filename. [This document section 3.3](http://msdn.microsoft.com/en-us/library/ms995843.aspx) says that this behaviour of sxs will only happen if the files are placed in the named directory, or otherwise they will be taken as shared.

invalidmagic's findings
-----------------------

i've been experimenting with nix on windows and mingw as a new toolchain:

[`https://invalidmagic.wordpress.com/2011/02/14/bringing-nix-to-its-limits/`](https://invalidmagic.wordpress.com/2011/02/14/bringing-nix-to-its-limits/)

two problems (discussed at the blog posting from above):

-   curl vs curl.exe seems to create problems (as cygwin sees both as the same program)
-   symlinks and dll files (PATH, missing LD_LIBRARY_PATH) is another

just don't forget to check it out! --[Invalidmagic](/User:Invalidmagic "wikilink") 19:49, 14 February 2011 (UTC)


test if comments work

yes, this is another line ;-)

--[Invalidmagic](/User:Invalidmagic "wikilink") 18:56, 15 February 2011 (UTC)

[Category:Nix](/Category:Nix "wikilink") [Category:Platforms](/Category:Platforms "wikilink") [Category:Windows](/Category:Windows "wikilink")