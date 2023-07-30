# A Beginner's Introduction to Nix

You can read the slides and find the links in [slides.md](slides.md).

You can play through the rendered slides in the boring way by browsing to the [html](html) folder.

You can play through the slides with reveal-md in a super-easy way with Nix, and it won't affect any existing software installed on your system:

* If you've not installed it already, I recommend the [The Determinate Nix Installer](https://zero-to-nix.com/concepts/nix-installer)
* Checkout this repo locally and cd into the repo
* Run `nix develop --command reveal-md -- slides.md -w`
* Profit!
