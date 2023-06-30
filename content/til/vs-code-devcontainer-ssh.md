---
title: "Frictionless SSH for VS Code Devcontainers on MacOS"
date: 2023-06-30
tags: 
  - vscode
  - ssh
  - macos
draft: false
---

## Introduction

I wanted to write down how I got a frictionless SSH setup for VS Code Devcontainers on MacOS, so I can e.g. interact with my Git(Hub) remote without additional steps like entering the SSH key's passphrase.

## Setup

### SSH Key

I try and create a separate SSH key pair for every use case. That means my `~/.ssh` dir contains a dedicated key for GitHub:

```shell
~ ls -1 .ssh/*github*
.ssh/id-github-ed25519
.ssh/id-github-ed25519.pub
```

The private key is secured by a passphrase.

### MacOS Keychain and SSH Agent

This will allow us to skip entering the passphrase when using the private key. First, we add the private key to MacOS' keychain (a.k.a. _Keychain Access.app_):

```shell
ssh-add --apple-use-keychain ~/.ssh/id-github-ed25519
```

(The `--apple-use-keychain` is a _relatively_ new option to `ssh-add`, but it should be available to you unless you're lagging behind several major OS releases.)

This step will do two things: It will 
1. add the key to `ssh-agent` (verify with `ssh-add -l`), and 
2. store the key's passphrase in the user's keychain.

The latter means it will also unlock your key automatically once you successfully logged in your operating system user.

Unfortunately, this command does not persist accross reboots. So, secondly, help the SSH Agent by adding the following line to your \.(zsh|bash|...)rc file:

```shell
ssh-add --apple-load-keychain &> /dev/null
```

### VS Code

With this setup, there's _nothing_ left to configure in VS Code itself, esp. in your `devcontainer.json` config file. Open the folder in your container and interact with whatever remote destination your SSH key is designated for. In my case, I can pull/push to GitHub.

Note: You might want to move the `ssh-add ...` commands into `devcontainer.json`'s `"initializeCommand"` option. I didn't want to do that, however, since it would make my container very much Apple-specific. With all my Devcontainers, I very much expect the OS setup to make the necessary SSH keys available to the "OS-agnostic" SSH Agent abstraction, which, in turn, automatically gets forwarded to the container when using Devcontainer's default setup.

