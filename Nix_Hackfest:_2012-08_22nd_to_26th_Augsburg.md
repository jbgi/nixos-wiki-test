---
title: Nix Hackfest: 2012-08 22nd to 26th Augsburg
permalink: /Nix_Hackfest:_2012-08_22nd_to_26th_Augsburg/
---

Leave a comment here if you are interested in attending a [hackfest](http://en.wikipedia.org/wiki/Hackathon) in Augsburg (or possibly Munich), Germany (6-9th July?) or participating remotely. Include your name or nick and mention what you would like to work on in particular. Try to include some detail if the topic requires an introduction for others, pointers to mailing list discussions, tickets etc. could also be helpful. We don't all have to work on the same thing, but it is good to take advantage of having people around to flesh things out.

goibhniu : Improve python support, PTS
garbas : Improve python support
yournick : your topic(s) of interest

Improve python support
----------------------

Work on solving some of the following issues

-   Python package impurity (stop packages from trying to install dependencies)
-   Run the package tests (currently most of them are disabled because they don't work)
-   Write nix tests which test that the python modules work (to some degree)
-   Look at generating nix expressions from pypi (see MarcWeber's work on this)

Package Tracking System
-----------------------

If I understand correctly the idea is to create some utilities to test the kwalitee of Nix package expressions, perhaps tied in with additional metadata. Aszlig mentioned it on IRC: <http://nixos.org/irc/logs/log.20120622>

<http://packages.qa.debian.org>

[Goibhniu](/User:Goibhniu "wikilink") 22:42, 28 June 2012 (UTC)