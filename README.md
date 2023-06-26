# Roman Böhm's Website

## What is it?

Website of Roman Böhm, containing, amongst others, blog posts and _Today I Learned_ bits. Setup with [Hugo](hugo.io/).

## How to try it out?

### On the Internet

Navigate to [https://www.romanboehm.com](https://www.romanboehm.com) or [https://romanboehm.github.io](https://romanboehm.github.io).

### Locally

1) Use VS Code to open the repo within the [Dev Container](https://code.visualstudio.com/docs/devcontainers) (or install Hugo on your device).
2) Run `hugo server -D` to serve the site locally.
2) Navigate to http://localhost:1313.

## Notes

For local development from within the Devcontainer: Make sure your host has an `ssh-agent` running and the SSH key for GitHub has been loaded into the agent (check with `ssh-add -l`); else you cannot interact with the remote repository.