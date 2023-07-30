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

Nix is a language, build system, package manager and Linux distribution (NixOS) with more than 80,000 packages.

I'm a recovering Mac user and "DevOpsy" guy with a love of declarative infrastructure. I am the beginner referenced in the title.

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
    nixpkgs.url = "github:NixOS/nixpkgs"; # also valid: "nixpkgs"
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

### Thank you

This meeting could have been a Git repo
https://gitlab.eng.roku.com/snewey/nix-intro

<hr>

Documentation: [Zero to Nix](https://zero-to-nix.com/)

Code: [nix-start-configs](https://github.com/Misterio77/nix-starter-configs)

Videos: [vimjoyer on YouTube](https://www.youtube.com/watch?v=bjTxiFLSNFA) | [Will T on YouTube](https://www.youtube.com/watch?v=QKoQ1gKJY5A&list=PL-saUBvIJzOkjAw_vOac75v-x6EzNzZq-&pp=iAQB)
