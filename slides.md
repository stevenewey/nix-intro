---
title: A Beginner's Introduction to Nix
theme: solarized
revealOptions:
    width: 1280
    height: 720
    transition: 'concave'
---

# A Beginner's Introduction to Nix

#### by Stephen Newey

#### 2023-08-03

---

## Nix and me

Nix is a language, build system and package manager with more than 80,000 packages. Available for Linux, macOS and WSL.

I'm a recovering Mac user and "DevOpsy" guy with a fondness for declarative infrastructure. I am the beginner referenced in the title.

---

## What's in it for me?

---

### Single use tools

```
[stephen@surfnix:~]$ nix-shell -p cowsay

[nix-shell:~]$ cowsay "What have you done for EBITDA today?"
 -------------------------------------- 
< What have you done for EBITDA today? >
 -------------------------------------- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

---

### Development environments

#### Off the shelf

```
[stephen@surfnix:~]$ nix develop "github:DeterminateSystems/zero-to-nix#python"
Welcome to a Nix development environment for Python!

[stephen@surfnix:~]$ python
Python 3.9.17 (main, Jun  6 2023, 09:45:13) 
[GCC 12.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

---

#### From a template

```bash
mkdir nix-javascript-dev && cd nix-javascript-dev
nix flake init --template "github:DeterminateSystems/zero-to-nix#javascript-dev"
```

---

## Imperative

<!-- .slide: data-transition="concave-in fade-out" -->

```Dockerfile
FROM node:lts-alpine

COPY package.json package-lock.json /app/

WORKDIR /app

RUN npm install --production

COPY . /app

EXPOSE 1948

WORKDIR /slides
ENTRYPOINT [ "node", "/app/bin/reveal-md.js" ]
CMD [ "/slides" ]
```


```bash
docker run --rm -p 1948:1948 -v /home/snewey/nix-intro:/slides webpronl/reveal-md
```

---

<!-- .slide: data-transition="fade-in concave-out" -->

## Declarative

```nix
{
  description = "A Beginner's Introduction to Nix";

  # Flake inputs
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs";
  };

  # Flake outputs
  outputs = { self, nixpkgs }:
    let
      # Systems supported
      allSystems = [
        "x86_64-linux" # 64-bit Intel/AMD Linux
        "aarch64-linux" # 64-bit ARM Linux
        "x86_64-darwin" # 64-bit Intel macOS
        "aarch64-darwin" # 64-bit ARM macOS
      ];

      # Helper to provide system-specific attributes
      forAllSystems = f: nixpkgs.lib.genAttrs allSystems (system: f {
        pkgs = import nixpkgs { inherit system; };
      });
    in
    {
      # Development environment output
      devShells = forAllSystems ({ pkgs }: {
        default = pkgs.mkShell {
          # The Nix packages provided in the environment
          packages = with pkgs; [
            nodejs-18_x # Node.js 18, plus npm, npx, and corepack
            nodePackages.reveal-md
          ];
        };
      });
    };
}
```

```bash
nix develop --command reveal-md slides.md
```

---

## Versioning

```json
{
  "nodes": {
    "nixpkgs": {
      "locked": {
        "lastModified": 1671636183,
        "narHash": "sha256-dboEYqb7vnH9pVEwgaWz7dzVi7eh6N5tRuhJ/nluoCg=",
        "owner": "NixOS",
        "repo": "nixpkgs",
        "rev": "60ff1ccd98a2f81347457a473c7a96b9b6166c88",
        "type": "github"
      },
      "original": {
        "owner": "NixOS",
        "repo": "nixpkgs",
        "type": "github"
      }
    },
    "root": {
      "inputs": {
        "nixpkgs": "nixpkgs"
      }
    }
  },
  "root": "root",
  "version": 7
}

```

----

```text
[stephen@surfnix:~/nix-intro]$ nix flake update
warning: updating lock file '/home/stephen/nix-intro/flake.lock':
• Updated input 'nixpkgs':
    'github:NixOS/nixpkgs/60ff1ccd98a2f81347457a473c7a96b9b6166c88' (2022-12-21)
  → 'github:NixOS/nixpkgs/d3a6adc9b6f11e260da168714ad8dcc938621610' (2023-07-30)

```

---

### What's actually going on here?


```
[nix-shell:~/nix-intro]$ type cowsay
cowsay is /nix/store/zbs7jdi6xg9g4vmzj7b3x8nz2pll55vh-cowsay-3.7.0/bin/cowsay
```

---

### Nix the build system

```nix
{ lib, stdenv, perl, fetchFromGitHub, fetchpatch, makeWrapper, nix-update-script, testers, cowsay }:

stdenv.mkDerivation rec {
  pname = "cowsay";
  version = "3.7.0";

  outputs = [ "out" "man" ];

  src = fetchFromGitHub {
    owner = "cowsay-org";
    repo = "cowsay";
    rev = "v${version}";
    hash = "sha256-t1grmCPQhRgwS64RjEwkK61F2qxxMBKuv0/DzBTnL3s=";
  };

  patches = [
    # Install cowthink as a symlink, not a copy
    # See https://github.com/cowsay-org/cowsay/pull/18
    (fetchpatch {
      url = "https://github.com/cowsay-org/cowsay/commit/9e129fa0933cf1837672c97f5ae5ad4a1a10ec11.patch";
      hash = "sha256-zAYEUAM5MkyMONAl5BXj8hBHRalQVAOdpxgiM+Ewmlw=";
    })
  ];

  nativeBuildInputs = [ makeWrapper ];
  buildInputs = [ perl ];

  postInstall = ''
    wrapProgram $out/bin/cowsay \
      --suffix COWPATH : $out/share/cowsay/cows
  '';

  makeFlags = [
    "prefix=${placeholder "out"}"
  ];

  passthru = {
    updateScript = nix-update-script { };
    tests.version = testers.testVersion {
      package = cowsay;
      command = "cowsay --version";
    };
  };

  meta = with lib; {
    description = "A program which generates ASCII pictures of a cow with a message";
    homepage = "https://cowsay.diamonds";
    changelog = "https://github.com/cowsay-org/cowsay/releases/tag/v${version}";
    license = licenses.gpl3Only;
    platforms = platforms.all;
    maintainers = with maintainers; [ rob anthonyroussel ];
  };
}
```

---

## Could you build a whole Linux distribution this way?

###### (Yes)

---

## NixOS

Your entire system, defined in `/etc/nixos/configuration.nix` &dagger;

<br>
<br>

&dagger; at least to start with

---

### configuration.nix

```nix
# Edit this configuration file to define what should be installed on
# your system.  Help is available in the configuration.nix(5) man page
# and in the NixOS manual (accessible by running ‘nixos-help’).

{ config, pkgs, ... }:

{
  imports =
    [ # Include the results of the hardware scan.
      ./hardware-configuration.nix
    ];

  # Bootloader.
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "surfnix"; # Define your hostname.
  # networking.wireless.enable = true;  # Enables wireless support via wpa_supplicant.

  # Configure network proxy if necessary
  # networking.proxy.default = "http://user:password@proxy:port/";
  # networking.proxy.noProxy = "127.0.0.1,localhost,internal.domain";

  # Enable networking
  networking.networkmanager.enable = true;

  # Set your time zone.
  time.timeZone = "Europe/London";

  # Select internationalisation properties.
  i18n.defaultLocale = "en_GB.UTF-8";

  i18n.extraLocaleSettings = {
    LC_ADDRESS = "en_GB.UTF-8";
    LC_IDENTIFICATION = "en_GB.UTF-8";
    LC_MEASUREMENT = "en_GB.UTF-8";
    LC_MONETARY = "en_GB.UTF-8";
    LC_NAME = "en_GB.UTF-8";
    LC_NUMERIC = "en_GB.UTF-8";
    LC_PAPER = "en_GB.UTF-8";
    LC_TELEPHONE = "en_GB.UTF-8";
    LC_TIME = "en_GB.UTF-8";
  };

  # Enable the X11 windowing system.
  services.xserver.enable = true;

  # Enable the GNOME Desktop Environment.
  services.xserver.displayManager.gdm.enable = true;
  services.xserver.desktopManager.gnome.enable = true;

  # Configure keymap in X11
  services.xserver = {
    layout = "gb";
    xkbVariant = "";
  };

  # Configure console keymap
  console.keyMap = "uk";

  # Enable CUPS to print documents.
  services.printing.enable = true;

  # Enable sound with pipewire.
  sound.enable = true;
  hardware.pulseaudio.enable = false;
  security.rtkit.enable = true;
  services.pipewire = {
    enable = true;
    alsa.enable = true;
    alsa.support32Bit = true;
    pulse.enable = true;
    # If you want to use JACK applications, uncomment this
    #jack.enable = true;

    # use the example session manager (no others are packaged yet so this is enabled by default,
    # no need to redefine it in your config for now)
    #media-session.enable = true;
  };

  # Enable touchpad support (enabled default in most desktopManager).
  # services.xserver.libinput.enable = true;

  # Define a user account. Don't forget to set a password with ‘passwd’.
  users.users.stephen = {
    isNormalUser = true;
    description = "Stephen Newey";
    extraGroups = [ "networkmanager" "wheel" ];
    packages = with pkgs; [
      firefox
    ];
  };

  # Allow unfree packages
  nixpkgs.config.allowUnfree = true;

  # List packages installed in system profile. To search, run:
  # $ nix search wget
  environment.systemPackages = with pkgs; [
    vim # Do not forget to add an editor to edit configuration.nix! The Nano editor is also installed by default.
  ];

  # Some programs need SUID wrappers, can be configured further or are
  # started in user sessions.
  # programs.mtr.enable = true;
  # programs.gnupg.agent = {
  #   enable = true;
  #   enableSSHSupport = true;
  # };

  # List services that you want to enable:

  # Enable the OpenSSH daemon.
  services.openssh.enable = true;

  # Open ports in the firewall.
  # networking.firewall.allowedTCPPorts = [ ... ];
  # networking.firewall.allowedUDPPorts = [ ... ];
  # Or disable the firewall altogether.
  # networking.firewall.enable = false;

  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It‘s perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/nixos/options.html).
  system.stateVersion = "23.05"; # Did you read the comment?

}
```

---

### Apply it now

```bash
nixos-rebuild switch
```

### Apply it on next boot
```bash
nixos-rebuild boot
```

---

### Generations

![NixOS Bootload Generations](images/nixos-grub.png)

---

### Get your git on

```bash
cd ~/nix-config
nixos-rebuild switch --flake .#surfnix
```

---

### Play hard mode

NixOS can boot with only `/nix` and `/boot`

https://github.com/nix-community/impermanence

```nix
{
  environment.persistence."/persistent" = {
    hideMounts = true;
    directories = [
      "/var/log"
      "/var/lib/bluetooth"
      "/var/lib/nixos"
      "/var/lib/systemd/coredump"
      "/etc/NetworkManager/system-connections"
      { directory = "/var/lib/colord"; user = "colord"; group = "colord"; mode = "u=rwx,g=rx,o="; }
    ];
    files = [
      "/etc/machine-id"
      { file = "/etc/nix/id_rsa"; parentDirectory = { mode = "u=rwx,g=,o="; }; }
    ];
    users.talyz = {
      directories = [
        "Downloads"
        "Music"
        "Pictures"
        "Documents"
        "Videos"
        "VirtualBox VMs"
        { directory = ".gnupg"; mode = "0700"; }
        { directory = ".ssh"; mode = "0700"; }
        { directory = ".nixops"; mode = "0700"; }
        { directory = ".local/share/keyrings"; mode = "0700"; }
        ".local/share/direnv"
      ];
      files = [
        ".screenrc"
      ];
    };
  };
}
```

---

### Take care of your home life

https://nix-community.github.io/home-manager/

```bash
cd nix-config
home-manager switch --flake .#stephen@surfnix
```

---

### Thank you

This meeting could have been a Git repo

https://gitlab.eng.roku.com/snewey/nix-intro

<hr>

Documentation: [Zero to Nix](https://zero-to-nix.com/)

Code: [nix-start-configs](https://github.com/Misterio77/nix-starter-configs)

Videos: [vimjoyer on YouTube](https://www.youtube.com/watch?v=bjTxiFLSNFA) | [Will T on YouTube](https://www.youtube.com/watch?v=QKoQ1gKJY5A&list=PL-saUBvIJzOkjAw_vOac75v-x6EzNzZq-&pp=iAQB)
