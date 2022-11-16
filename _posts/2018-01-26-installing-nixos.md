---
title: "Installing NixOS"
date: "2018-01-26"
categories: 
  - "linux"
---

This is just a small introduction to a Linux distribution that's been getting traction lately, especially in functional programming ecosystems.

# NixOS: The purely functional Linux distribution

NixOS is a really cool idea: the OS and its various components and apps is fully determined by its configuration, in a deterministic fashion. In other words, with NixOS systems become stateless and immutable. _Nix_ is also the language that is used to express this configuration: it is a pure functional and lazy DSL, designed for system configuration. Being a functional and declarative language, it allows building abstractions for configuration and composable bricks that can be assembled to produce a certain environment.

## `/etc/nixos/configuration.nix`

The entire system is determined by means of the `configuration.nix` file. This is the root file for the configuration. Nix has an import mechanism, so configurations can be organized in multiple files and folders like real software projects. Here a example of a very simple `configuration.nix` file which defines a minimal system:

```nix
{ config, lib, pkgs, ... }:
{
  boot.loader.grub.device = "/dev/sda";
  fileSystems."/".device = "/dev/sda1";

  networking.firewall = {
    enable = true;
    allowedTCPPorts = [ 80 ];
  };

  environment.systemPackages = with pkgs; [
    wget
    git
  ];

  services = {
    sshd.enable = true;
  };
}
```

The `{ arg1, arg2, ..} : {}` notation is how you define functions in the Nix language. Note how the system configuration itself is just a function!

# Installation

The installation steps I went through are documented in this [github repo](https://github.com/jchapuis/nixos-configuration). This repository is also where I store all my personal nix configuration files to restore machines or grab elements of configuration.

## Reproducibility

In a nutshell, with NixOS installing a fully-configured system boils down to:

1. bootstrapping it with the nixos livecd or usb key
2. downloading (or editing) the configuration
3. running the `nixos-install` command

Improvements and changes to system configuration can be tracked with version control, and easily propagated to all the machines via `nixos-rebuild`.

With NixOS, it's never been easier to find oneself at home in any environment!

# Links

- Helpful personal installation procedures [here](https://chris-martin.org/2015/installing-nixos) and [here](https://github.com/jluttine/nixos-configuration)
- [ScalaIO talk](https://www.youtube.com/watch?v=YWSeJQKWw9g) on NixOs
- [Configuration options reference](https://nixos.org/nixos/manual/options.html)
- [Searchable packages directory](https://nixos.org/nixos/packages.html#idea)
- [Cheat sheet](https://nixos.wiki/wiki/Cheatsheet)
- [Wiki](https://nixos.wiki/)
