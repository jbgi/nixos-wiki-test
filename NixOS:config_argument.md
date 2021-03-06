---
title: NixOS:config argument
permalink: /NixOS:config_argument/
---

This argument gives access to the entire configuration of the system. It is computed from option declarations and option definitions defined inside all modules used for the system.

Simple Case
-----------

The following code is used to support the explanation.

    {config, pkgs, ...}:

    {
      options = {
        foo = pkgs.lib.mkOption {
          description = "...";
        };
      };

      config = {
        bar = config.foo;
      };
    }

This snippet of code is a module which declare the option "`foo`" and define the option "`bar`". The option "`bar`" is defined with the value `config.foo`. This attribute `foo` of the `config` argument, is the result of the [evaluation](/http://wiki.nixos.org/wiki/NixOS:Declaration#Evaluation "wikilink") of the definitions and the declarations of the option "`foo`". Definitions of the option "`foo`" may exists in other module used by the system.

Adding additional modules to the system may change the value of `config.foo` and may change the behavior of the previous module.

Conditional Statements
----------------------

The process of module computation is highly recursive and may cause trouble when you want to add control flow statements. A common mistake is to use "`if`" or "`assert`" statements in the computation of a module.

    {config, pkgs, ...}:

    {
      options = {
        foo = pkgs.lib.mkOption {
          default = false;
          type = with pkgs.lib.types; bool;
          description = "enable foo";
        };
      };

      config =
        if config.foo then
          { bar = 42; }
        else
          {};
    }

The previous module cause an infinite loop.

... explanation coming soon ...