---
title: Open issues:ways of updating sources automatically
permalink: /Open_issues:ways_of_updating_sources_automatically/
---

purpose: jumping to homepages getting source and name of the latest version takes time. So we want to automate this.

Like Tobias Hunger said (and maybe someone else before) we may want to consider learning or utilizing uscan. (Link?)

The following implementations exist:

Ludovic Courtes eval xml then parse
-----------------------------------

For full information read the original thread [mailinglist](http://www.mail-archive.com/nix-dev@cs.uu.nl/msg03825.html)

Summary: nix-instantiate evaluates nixpkgs. The resulting (huge) XML is parsed by a scheme helper application finding gnu packages and updating their sources. Eelco Dolstra argued that the update information could be put into the meta description so that more ways of updating packages can be added. It currently updates gnu packages only (?)

Marc Weber's comments: The idea is great. However it depends on evaluation of nixpkgs which can be different depending on configuration options. Also if you put update information into the meta attributes which are not passed to the builder there are many lines between the sorce declaration and the update information. So its easy to miss.

Michael Raskins autoupdate scripts
----------------------------------

description: pkgs/build-support/upstream-updater/design.txt

Summary: The update information which basically consists of url of html page containing update links a regex extracting them is put into a .nix file. A shell script then evaluates the .nix expression gathering that data, downloading the page finding the latest version. The updated information is then put in yet another file which is imported by default.nix. Example: webkit.

Marc Weber's comments: I dislike many small files. When merging branches or deleting packages they cause more work. Also many small files take longer to seek on non SSD disks and fill up more disk space ? I'm not sure those are strong arguments. I've put a clone of the script into my nixpkgs-utilities (gitorious) repository adding a --help option.

AFAIK Eelco Dolstra also dislikes having many small files for this purpose.

Marc Weber's nix-repository-manager
-----------------------------------

[github](http://github.com/MarcWeber/nix-repository-manager) basic idea: Keep it simple. Text regions are introduced which start by special markers which are commented lines. The comments contain information about how to update the region in a Nix attr like format. An example is HaXe. There are two phases for updating a package:

(1) clone /update remote (git,hg,...) repository into a local directory and create .tar.gz for testing

(2) upload .tar.gz publishing it

Its you choosing between using the testing or published version by configuration option.

Example taken from HaXe:

`   src_haxe_swflib = {`
`     # REGION AUTO UPDATE:                                { name = "haxe_swflib"; type="cvs"; cvsRoot = ":pserver:anonymous@cvs.motion-twin.com:/cvsroot"; module = "ocaml/swflib"; groups = "haxe_group"; }`
`     src = sourceFromHead "haxe_swflib-F_10-43-46.tar.gz"`
`                  (fetchurl { url = "`[`http://mawercer.de/~nix/repos/haxe_swflib-F_10-43-46.tar.gz`](http://mawercer.de/~nix/repos/haxe_swflib-F_10-43-46.tar.gz)`"; sha256 = "a63de75e48bf500ef0e8ef715d178d32f0ef113ded8c21bbca698a8cc70e7b58"; });`
`     # END`
`   }.src;`

Marc Weber's comments: I wrote it and I still prefer it because the update information is nearby the source so you can't miss it. However care should be taken that the contents of the regions don't exceed a couple of lines so that its easy to see that its a region. That .tar.gz files are created (and not fetchGit etc) are used can be controversal:

+ fast download, always works (unless my mirror for those .tar.gz packages goes down)

+ fast updates because VCS updates are incremental

+ supports hack-nix

- its harder to verify that I didn't manipulate the source packages - so you have to trust me.

I plan to support Michael Raskins feature in the near future getting links by xpath from HTML pages selecting the newest version. It can be installed easily using the [nixpkgs-haskell-overlay](http://github.com/MarcWeber/nixpkgs-haskell-overlay)

Tobias Hunger's Auto package updating script v2
-----------------------------------------------

[mailinglist first version](http://article.gmane.org/gmane.linux.distributions.nixos/5575/match=auto+package+updating+script)

[mailinglist v2, based on Ludo's ideas?](http://thread.gmane.org/gmane.linux.distributions.nixos/5594/focus=5597)

Marc Weber's comments: None. I haven't had time to look into it. v2 also uses evaluation so it also suffers from the configuration issue.

Tobias Hunger's comments: The script extracts the following attributes from the XML:

-   name (for name and version information)
-   watchfile (for a debian/watch file to use for the actual updating process as well as the .nix file name to update)
-   src/url\[s\] (in case we want to update the URL)
-   src/hash value and type (to replace the hash later)

There is no extra update information is needed inside the nix expressions. It seems to work reasonably well with the files I treid to convert so far (about 15 or so).

The watch file is an extra small file of course, but having it enables me to use debian's uscan utility to actually figure out all the update information instead of reinventing the wheel. Uscan is widely used inside debian and has special magic to support sites like sourceforge, etc. and even can be used to extract package information from download pages meant for humans (e.g. bzip2 or zlib).

Status so far:

-   Extracting information from the nix expressions works great
-   Finding updates is very solid
-   Updating the nix expression is still not perfect, especially with files wile the one found in pkgs/server/x11/xorg which has lots of derivations defined in it. There still is a chance that my script ends up replacing the wrong things there. I need to find a way to improve that.
