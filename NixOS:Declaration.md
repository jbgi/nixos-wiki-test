---
title: NixOS:Declaration
permalink: /NixOS:Declaration/
---

NixOS module has declarations to provide an interface for other modules. Option declarations are used to make NixOS aware of configuration possibilities or to hide the complexity of configuration. Option declarations are used to provide an interface over the inherent complexity of configuring the system.

Option attributes
=================

Options are declared with the `mkOption` function which is provided by the Nix package collection library (`pkgs.lib` attribute set). An option declaration may contain the following attributes:

-   `name` `(string)`: This attribute is by default set to the location of the option. The name is used in error messages and in the generated manual.
-   `default` `('t)`: The value used when no definitions are provided.
-   `example` `('t)`: The value used as example to show a possible configuration.
-   `description` `(string)`: A multi-line text which describe in a few sentences what is achieved by the option.
-   `type` `(/'t/)`: The expected type of the default value and the option definitions. The type may provide default code for `merge` and `check` attribute.
-   `merge` `('t list -> 't)`: A function used to merge all definitions into one result which has the same type.
-   `apply` `('t -> any)`: A function used to provide an abstraction over the merged result. This function is mostly used when an usual transformation is made of the result.
-   `options` `(module list)`: This attribute contains a list of modules when the type allow to have embedded modules.

An option can be declared in multiple modules, with one condition which is that no attribute collisions exists between the declarations except for the `options` attribute.

Evaluation
----------

The evaluation is done with the following piece of code where `opt` is the option declaration. You can find this piece of code inside \[<https://svn.nixos.org/repos/nix/nixpkgs/trunk/pkgs/lib/modules.nix>|`pkgs/lib/modules.nix`\].

      opt.apply (
        if isNotDefined then
          if opt ? default then opt.default
          else throw "Not defined."
        else opt.merge definitions
      )

The `apply` function is replaced by the identity function and the `merge` function is replaced by the `mergeDefaultOption` function if they are not defined.

Types
=====

Types are used to ensure the correctness of definitions and to provide a way to merge the definitions. All common types are available in \[<https://svn.nixos.org/repos/nix/nixpkgs/trunk/pkgs/lib/types.nix>|`pkgs/lib/types.nix`\]. To declare a type you have to write something like:

    services.fooBar.option = pkgs.lib.mkOption {
      type = with pkgs.lib.types; uniq string;
      description = "
        ...
      ";
    };

The `check` function provided by the type can be a source of strictness, but this is rare because most of the time checked option definitions are fully used by the merge function.

Simple Types
------------

-   `inferred`: Useful when it is used under a meta-type.
-   `bool`: A Boolean useful for enable flags. The merge function is a logical OR between all definitions.
-   `int`: An Integer.
-   `string`: A string where all definitions are concatenated.
-   `envVar`: A string where all definitions are concatenated with a colon between all definitions.
-   `attrs`: An attribute set. (similar to `attrsOf inferred`)
-   `package`: A derivation.

Meta Types
----------

Data meta-types:

-   `listOf t`: A list of elements with the type `t`.
-   `attrsOf t`: An attribute set of elements with the type `t`. The merge function zip all attribute sets into one. Attribute values of the resulting attribute set are merged with the merge function of the type `t`.

Definition meta-types:

-   `uniq t`: This type define raise an error if more than one definitions exists. All other properties are inherited from the type `t`. This is useful to avoid ambiguous definitions.
-   `none t`: This type define raise an error if at least one definitions exists. All other properties are inherited from the type `t`. This is useful to provide a computation result to other modules. See also the `apply` function of option declarations.

Module Type
-----------

-   `optionSet`: This type is used to benefit from the modular system used by NixOS inside an option. When this type is enabled, it merge the `options` attribute of option declarations with each definition. The result is the fixed configuration of each module. This option is often see in conjunction with `attrsOf` or `listOf`. Modules declared in option declaration appear in the generated manual with a prefix representing the container type, respectively "`.<name>`" and "`.*`". Reference to definition files do not appears in the generated manual because the system cannot track them.
