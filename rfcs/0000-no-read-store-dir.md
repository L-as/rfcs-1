---
feature: no-read-store-dir
start-date: 2021-07-03
author: Las Safin
co-authors:
shepherd-team:
shepherd-leader:
related-issues:
---

# Summary
[summary]: #summary

Set the permissions for /nix/store to 1771 instead of 1775, disabling reading the directory for other users.

# Motivation
[motivation]: #motivation

This means that you can not trivially see all the paths in the nix store, e.g. `ls` won't work
on /nix/store without sudo unless you're in the nixbld group.

Almost everything in NixOS needs access to /nix/store, which means that all your systemd services,
your flatpak programs, your manually [bubblewrap](https://github.com/containers/bubblewrap)ed programs
will almost certainly have access to /nix/store and see all of its contents.
That means they will see your NixOS configuration, initrd secrets (unless your bootloader has support for it),
all the programs you've built, your usage activity, and possibly also your important secrets if you
(unfortunately) had to put them in the store!

By simply removing the ability to read /nix/store (but not execute!), programs will only be able
to access the store paths *which they already know the hash and name of*. This is a huge boon to security
even if you don't have secrets in your store, as it will almost completely eliminate the above
problems. That isn't to say it's a complete solution to security, but it will *allow* much more
complete solutions to security, for example with bubblewrap, meaning that you can be sure that
your sandboxed game won't suddenly scrape your store and send it off to NSA.

# Detailed design
[design]: #detailed-design

Set the 1775
in [nixpkgs/nixos/modules/system/boot/stage-2-init.sh](https://github.com/NixOS/nixpkgs/blob/8284fc30c84ea47e63209d1a892aca1dfcd6bdf3/nixos/modules/system/boot/stage-2-init.sh#L62),
in [nix/scripts/install-multi-user.sh](https://github.com/NixOS/nix/blob/cf1d4299a8fa8906f62271dcd878018cef84cc30/scripts/install-multi-user.sh#L577), and
in [nix/src/libstore/globals.hh](https://github.com/NixOS/nix/blob/ba8b39c13003c8ddafb6bec308997e09b9851c46/src/libstore/globals.hh#L278)
to 1771.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

Losing the read (r) bit means that you can't list the files inside the store.
The execute (x) bit allows us to `cd` to it and also access paths inside the store.

E.g. `ls "$(readlink /nix/var/nix/profiles/system)"` will still work, since this is a directory
inside the store, and not the store itself, but you can't without sudo do `ls /nix/store` to find your system configuration.

# Drawbacks
[drawbacks]: #drawbacks

It might be a slight annoyance since shell completion won't work in the /nix/store anymore, e.g.
if you have some hash 48914, you can't type `/nix/store/48914<tab>` to get the full path anymore.

# Alternatives
[alternatives]: #alternatives

I had [this script](https://github.com/L-as/nix-misc/blob/e844a03ebf4cad4fc8eca0e52306788b70c2a60d/claybox.rb) that is a wrapper around bubblewrap before, but doing this
is a lot cleaner, is system-wide, and is a lot faster, since bind mounting each individual store path with bubblewrap can be quite slow
if you have many (it's O(n^2) IIRC).

```nix
system.activationScripts.chmod-store.text = ''
  ${pkgs.util-linux}/bin/unshare -m ${pkgs.bash}/bin/sh -c '${pkgs.util-linux}/bin/mount -o remount,rw /nix/store ; ${pkgs.coreutils}/bin/chmod 1771 /nix/store'
'';
```
can be used at the moment without this becoming standard, but it being standardized will mean it won't break due to upstream changes, and more users will
see its benefit.
In addition, `nix-env` sets it back to 1775, which means for the above to work you must prevent use of nix-env completely.

# Unresolved questions
[unresolved]: #unresolved-questions

There doesn't seem to be any.

# Future work
[future]: #future-work

There could be more work in the future toward properly sandboxing systemd services and such by default,
which could also make use of this vital change.
