---
title: "Frictionless SSH for VS Code Dev Containers on MacOS"
date: 2023-06-30
tags: 
  - vscode
  - ssh
  - macos
draft: false
---

## Introduction

I wanted to write down how I got a frictionless SSH setup for [VS Code Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) on MacOS where I have the necessary keys available, but don't need to manually unlock them anymore. I can e.g. simply push to GitHub/pull from GitHub from my dev environment inside the container.

## Setup

### SSH Key(s)

I usually create a separate SSH key pair for every use case. That means my `~/.ssh` dir contains a dedicated key for GitHub, which I'll use as an example in the remainder of the post:

```shell
~ ls -1 .ssh/*github*
.ssh/id-github-ed25519
.ssh/id-github-ed25519.pub
```

The private key is also secured by a passphrase.

### MacOS Keychain and SSH Agent

The VS Code Dev Containers extension facilitates SSH by automatically "forwarding" (for lack of a better word) the host system's SSH Agent to the Dev Container. Now, this means that integrating the Mac OS keychain and the SSH Agent will allow us to skip entering the passphrase when using the private key _inside_ the container. This is done by two commands:

First, we need to add the private key to MacOS' keychain (a.k.a. _Keychain Access.app_):

```shell
ssh-add --apple-use-keychain ~/.ssh/id-github-ed25519
```

(The `--apple-use-keychain` and its `--apple-load-keychain` sibling are _relatively_ new options to `ssh-add`, but they should be available to you unless you're lagging behind several major MacOS releases.)

This step will do two things: It will 
1. add the key to `ssh-agent` (verify with `ssh-add -l`), and 
2. store the key's passphrase in the user's keychain.

The latter means it will also unlock your key automatically once you've successfully logged in your operating system user.

Unfortunately, this command does not persist accross reboots. So, secondly, we need to help the SSH Agent here by adding the following line to our `.zshrc` or `.bashrc` or whatever-is-doing-bootstrapping file:

```shell
ssh-add --apple-load-keychain &> /dev/null
```

### VS Code

With this setup, there's _nothing_ left to configure in VS Code itself, esp. in your `devcontainer.json` config file. Open your workspace in your container and interact with whatever remote destination your SSH key is designated for.

Note: You may move the `ssh-add ...` commands directly into `devcontainer.json`'s `"initializeCommand"` option. I didn't want to do that, however, since it would make my container Apple-specific. This goes against my philosphy of keeping Dev Containers OS-agnostic. I very much expect to have whatever OS I'm using configured to make the SSH keys available to the SSH Agent abstraction, not VS Code look for the keys itself.

