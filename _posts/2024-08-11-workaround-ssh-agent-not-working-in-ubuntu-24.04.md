---
layout: post
title: Workaround ssh agent not working in Kubuntu 24.04
date: 2024-08-09
author: Fxzjshm
category: Linux
tags: [Linux]
---

After upgrading from 22.04 to 24.04 through `do-release-upgrade`,
GNOME keyring no longer manages `ssh` key passwords when using Plasma.
(Still working on GNOME).

<!-- wrong: cannot fix -->

<!-- Related change is <https://gitlab.gnome.org/GNOME/gnome-keyring/-/commit/25c5a1982467802fa12c6852b03c57924553ba73> ("build: Remove build with ssh component from default build instructions") (from <https://wiki.archlinux.org/title/GNOME/Keyring#SSH_keys>).

As said in commit message, enable gcr:

```bash
systemctl enable --now gcr-ssh-agent --user
```

Notice this operation requires `--user`, otherwise it cannot find `gcr-ssh-agent.service`. -->

After some digging, it is noticed that `SSH_ASKPASS` is set to `/usr/bin/ksshaskpass` and didn't find where it is set; however ssh-askpass isn't working properly.

As said in <https://wiki.archlinux.org/title/KDE_Wallet#Using_the_KDE_Wallet_to_store_ssh_key_passphrases>,
will use this script in `.bash_profile` as a workaround:

```bash
# https://wiki.archlinux.org/title/KDE_Wallet#Using_the_KDE_Wallet_to_store_ssh
if [ "$SSH_ASKPASS" == "/usr/bin/ksshaskpass" ]; then
  export SSH_ASKPASS_REQUIRE=prefer
fi
```

<!-- more -->