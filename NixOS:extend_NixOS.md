---
title: NixOS:extend NixOS
permalink: /NixOS:extend_NixOS/
---

This page contains a tutorial to get major points about NixOS configuration. To illustrate this tutorial we use the example of a user who wants to start an irc client each time his computer is starting. This page will describe all evolutions of the user configuration files.

Simple Configuration
====================

The user wants to start his irc session inside NixOS. In other distribution, you will add an init script inside `/etc/init.d` or any other similar directory. In NixOS, you have to keep your dependencies clean, this implies that you have to use your configuration to register the init script.

To start the irc session, let it be inside a screen daemon with irssi. The user `/etc/nixos/configuration.nix` will become something like:

    {pkgs, ...}:

    # pkgs is used to fetch screen & irssi.

    {
      jobs.ircSession = {
        description = "Start the irc client of username."
        startOn = "started network-interfaces";
        exec = ''su -c "${pkgs.screen}/bin/screen -m -d -S irc ${pkgs.irssi}/bin/irssi" username'';
      };

      environment.systemPackages = [ pkgs.screen ];

      # ... usual configuration ...
    }

The user use this expression on his server and use an alias on the command

    ssh username@my-server -t screen -d -R irc

to log on his irc session.

Conditions
==========

Now the user has another computer similar to his server. He wants to re-use his server configuration, because most of the hardware is similar. To avoid duplicating everything, he decide to use the host name of the computer to enable or disable services. He rewrites his `configuration.nix` file with the `mkIf` property.

    {config, pkgs, ...}:

    {
      jobs = pkgs.lib.mkIf (config.networking.hostname == "my-server") {
        ircSession = {
          description = "Start the irc client of username."
          startOn = "started network-interfaces";
          exec = ''su -c "${pkgs.screen}/bin/screen -m -d -S irc ${pkgs.irssi}/bin/irssi" username'';
        };
      };

      environment.systemPackages = pkgs.lib.mkIf (config.networking.hostname == "my-server") [ pkgs.screen ];

      # ... usual configuration ...
    }

This code seems to fit his expectations, but many conditions have to be introduced and this is becoming harder to maintain it when he wants to change his server host name.

Multiple Configurations
=======================

To avoid the complexity of the previous mixing done inside his `configuration.nix` file. He decides to split his configuration into multiple file where each concerns are separated. He keeps the configuration.nix file as his general configuration for both computers and factor the rest inside a file named `irc-client.nix`.

The content of his `configuration.nix` file becomes:

    {
      require = [
        ./irc-client.nix
      ];

      # ... usual configuration ...
    }

and the content of the `irc-client.nix` file becomes:

    {config, pkgs, ...}:

    pkgs.lib.mkIf (config.networking.hostname == "my-server") {
      jobs.ircSession = {
        description = "Start the irc client of username."
        startOn = "started network-interfaces";
        exec = ''su -c "${pkgs.screen}/bin/screen -m -d -S irc ${pkgs.irssi}/bin/irssi" username'';
      };

      environment.systemPackages = [ pkgs.screen ];
    }

This way, the complexity of changing the condition is reduced. In addition, he can discover consistency issue within the `irc-client.nix` file which makes this file maintainable.

Sharing Configuration
=====================

The user has discuss a bit of his configuration on irc and some other person wants to benefits form his modification. Thus he has to remove all parts which are dependent on his system and make it more general. So he decides to replace the condition and the username by options.

    {config, pkgs, ...}:

    let
      cfg = config.services.ircClient;
    in

    with pkgs.lib;

    {
      options = {
        services.ircClient = {
          enable = mkOption {
            default = false;
            type = with types; bool;
            description = ''
              Start an irc client for an user.
            '';
          };

          user = mkOption {
            default = "username";
            type = with types; uniq string;
            description = ''
              Name of the user.
            '';
          };
        };
      };

      config = mkIf cfg.enable {
        jobs.ircSession = {
          description = "Start the irc client of ${cfg.username}."
          startOn = "started network-interfaces";
          exec = ''su -c "${pkgs.screen}/bin/screen -m -d -S irc ${pkgs.irssi}/bin/irssi" ${cfg.user}'';
        };

        environment.systemPackages = [ pkgs.screen ];
      };
    }

This module is now independent of the system and the user can modify his `configuration.nix` to get his previous configuration.

    {config, ...}:

    {
      require = [
        ./irc-client.nix
      ];

      services.ircClient.enable = config.networking.hostname == "my-server";
      services.ircClient.user = "username";

      # ... usual configuration ...
    }

Multiple Daemons
================

The user now host multiple person on his server, and they want to have the same irc client running in background. One easy possibility would be to replace the user name by a list of user names, but this would not add more value in this tutorial. Another solution is to extend jobs with the irc client options. This will extend the options available inside `jobs.<name>`.

    {config, pkgs, ...}:

    let
      anyIrcClient = with pkgs.lib;
        fold (j: v: v || j.ircClient.enable) (attrValues config.jobs);
    in

    with pkgs.lib;

    {
      options = {
        jobs.options = {config, ...}: let
          cfg = config.ircClient;
        in {
          options = {
            ircClient.enable = mkOption {
              default = false;
              type = with types; bool;
              description = ''
                Start an irc client for an user.
              '';
            };

            ircClient.user = mkOption {
              default = "username";
              type = with types; uniq string;
              description = ''
                Name of the user.
              '';
            };
          };

          config = mkIf cfg.enable {
            description = "Start the irc client of ${cfg.username}."
            startOn = "started network-interfaces";
            exec = ''su -c "${pkgs.screen}/bin/screen -m -d -S irc ${pkgs.irssi}/bin/irssi" ${cfg.user}'';
          };
        };
      };

      config = mkIf anyIrcClient {
        environment.systemPackages = [ pkgs.screen ];
      };
    }

... work in progress ...