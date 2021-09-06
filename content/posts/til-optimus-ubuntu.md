+++
title = "Ubuntu and Nvidia Optimus"
date = "2019-03-10"
tags = [
    "til",
    "linux"
]
draft = false
+++

How to: Make Nvidia Optimus work after installing Ubuntu 21.04.

I own a Thinkpad T480 with a dedicated GPU (Nvidia Geforce MX 150) in addition to the onboard Intel GPU. The easiest way to have a Linux distribution with full Optimus support running would be to download the most recent PopOS! image with the proprietary Nvidia drivers baked in.

Since I cannot stand some of the modifications System76 made to Gnome I just want to get the most recent Ubuntu (or Kubuntu) version and go from there. Ubuntu installed just fine without any graphics glitched or black screens and `prime-select query` shows _nvidia_. Something was not right, however: `nvidia-settings` produced a tiny, empty window and judging from `powertop` stats it seemed like the system still used the onboard intel graphics.

After digging through ten year-old Stack Overflow posts and "solved" topics on the Nvidia forums I tried the most straightforward suggestion, which was to update to a more recent version of the Nvidia driver than what was provided through the Ubuntu install. So the whole endeavor might now be summarized as 
0. Install Ubuntu with the "Install 3rd party software" option enabled.
1. Run `sudo apt install nvidia-driver-470` (or whatever is newest/stable enough).

Note: If you need an even newer version you might want to add the _graphics-drivers/ppa_ repository and run `sudo apt update` before `install`.



