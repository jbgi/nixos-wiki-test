---
title: Nix Runtime Environment Wrapper
permalink: /Nix_Runtime_Environment_Wrapper/
---

Some programs need information which is only available at runtime, such as dynamically linked libraries, files, or environment variable values. How are these libraries found? This problem is solved differently by different software projects. Because Nix is a non-standard system, When an application is packaged to run in a Nix system, it may not be able to find some things at runtime.

Because most programs packaged by Nix require a slightly modified build process, Nix packages include attributes which explicitly ask for build-time customizations. Unlike build-time customizations, modifying an application's runtime environment is relatively uncommon. As a result, a Nix package does not explicitly prompt a package maintainer to describe them. Lacking native support of runtime modifications in a Nix package, the Nix community uses a shell script named `makeWrapper` to solve this problem. Documentation for this package follows.

MakeWrapper Package
-------------------

The [`makeWrapper` package](https://github.com/NixOS/nixpkgs/blob/41840af6894bf718a1038ba1045adef26a687919/pkgs/build-support/setup-hooks/make-wrapper.sh) adds a shell function, `wrapProgram`, which will ensure the specified program has the specified environment when it is executed. This package consists of a single shell script, [`makeWrapper.sh`](https://github.com/NixOS/nixpkgs/blob/41840af6894bf718a1038ba1045adef26a687919/pkgs/build-support/setup-hooks/make-wrapper.sh).

To use the `makeWrapper` function in a package, first include the `makeWrapper` package in the `buildInputs` attribute. Then, use the `makeWrapper` shell function somewhere in the build process, such as in the "postInstall" phase. For documentation on these functions and examples, see below.

Shell Functions
---------------

<strong>`wrapProgram(String programPath, List<String> environmentExpressions)`</strong>

Rename the executable file at the specified program path and create a wrapper program in its place which executes the original executable file in an environment which is modified based on the specified environment expressions.

The specified file is renamed by prepending a dot "." and appending "-wrapped" to the original file name. For example, "myprog" will be renamed to ".myprog-wrapped".

Each environment expression consists of an operator and one, two, or three arguments, depending on the operator. The supported operators are documented on this wiki page.

<strong>`makeWrapper(String originalProgramPath, String wrapperScriptPath, List<String> environmentExpressions)`</strong>

Build an executable shell script at the specified wrapper path which uses the specified environment expressions to modify the current shell environment before executing the specified original program.

Each environment expression consists of an operator and one, two, or three arguments, depending on the operator. The supported operators are documented on this wiki page.

<strong>`addSuffix(String suffix, List<String> values)`</strong>

Append the specified suffix to each of the specified values.

<strong>`filterExisting(List<String> filePaths)`</strong>

Remove all files from the specified file paths which do not exist.

### Environment Expression Operators and Arguments:

Each environment expression consists of an operator and one, two, or three arguments, depending on the operator.

<strong>`--set envVarName varValue`</strong>

Set the specified environment variable to the specified value.

<strong>`--run command`</strong>

Execute the specified command.

<strong>`--suffix envVarName separator value`</strong>

Append the specified value to the specified environment variable, placing the specified separator between the old and new values.

<strong>`--prefix envVarName separator value`</strong>

Prepend the specified value to the specified environment variable, placing the specified separator between the old and new values.

<strong>`--suffix-each envVarName separator values`</strong>

Append the specified values to the specified environment variable, placing the specified separator between each old and new value.

<strong>`--suffix-contents envVarName separator fileNames`</strong>

Append the contents of the specified files to the specified environment variable, placing the specified separator between the contents of each file.

<strong>`--prefix-contents`</strong>

Prepend the contents of the specified files to the specified environment variable, placing the specified separator between the contents of each file.

<strong>`--add-flags flag`</strong>

Use the specified flags when executing the original program.

<em>`(remaining arguments)`</em>

All remaining arguments are considered flags to use when executing the original program.

Examples
--------

The [KMPlayer package](https://github.com/NixOS/nixpkgs/blob/f305fe1c02b223463af9263a9800fc6b606527db/pkgs/applications/video/kmplayer/default.nix) uses `wrapProgram` to ensure the package's `bin` directory is in the `PATH` variable when it is executed.

        postInstall = ''
          wrapProgram $out/bin/kmplayer --suffix PATH : ${mplayer}/bin
        '';

The [Duplicity package](https://github.com/NixOS/nixpkgs/blob/80452891996797c54677ca16ca9a8caea2d11e72/pkgs/tools/backup/duplicity/default.nix) uses multiple environment expressions.

        installPhase = ''
          python setup.py install --prefix=$out
          wrapProgram $out/bin/duplicity \
            --prefix PYTHONPATH : "$(toPythonPath $out):$(toPythonPath ${boto})" \
            --prefix PATH : "${gnupg}/bin:${ncftp}/bin"
          wrapProgram $out/bin/rdiffdir \
            --prefix PYTHONPATH : "$(toPythonPath $out):$(toPythonPath ${boto})" \
        '';

The [Firefox Plugin Wrapper](https://github.com/NixOS/nixpkgs/blob/41840af6894bf718a1038ba1045adef26a687919/pkgs/applications/networking/browsers/firefox/wrapper.nix) demonstrates advanced useage by using the `makeWrapper` packages's utility functions: `addSuffix` and `filterExisting`.

        makeWrapper "${browser}/bin/${browserName}" \
            "$out/bin/${browserName}${nameSuffix}" \
            --suffix-each MOZ_PLUGIN_PATH ':' "$plugins" \
            --suffix-each LD_LIBRARY_PATH ':' "$libs" \
            --suffix-each LD_PRELOAD ':' "$(cat $(filterExisting $(addSuffix /extra-ld-preload $plugins)))" \
            --prefix-contents PATH ':' "$(filterExisting $(addSuffix /extra-bin-path $plugins))"