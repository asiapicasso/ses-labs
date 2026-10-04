# SeS-Labs

## Lab 1 : Initial Setup

### Part 1 : Install the development environment for the target system

- The various software to install and configure are:
- Git
- Docker
- Visual Studio Code (VSCode) along the Dev Container extension
- balenaEtcher
- Serial communication
- Prepare the containerized development environment for the target system

### Part 2 : Generate a vanilla SD card image, then a custom one

- Configure Buildroot for the Raspberry Pi 4
- Generate a vanilla SD card image using Buildroot
- Inspect the image
- Flash the image to a SD card
- Connect the target system to the host
- Successfully boot Linux and log into a shell

### Part 3 : Create your own config and generate a custom SD card image with U-Boot support

- Create your own Buildroot configuration
- Modify your config to change the hostname and set a root password
- Modify your config to use U-Boot instead of the proprietary firmware
- Boot into U-Boot

## Lab 2 : U-Boot and booting Linux

In this lab, you’ll go through the following points:

- Configure U-Boot to boot the Linux kernel
- Inspect and modify U-Boot configuration
- Useful U-Boot variables & memory layout
- Change U-Boot’s runtime behavior
- Establish a network connection between host and target systems
- Kernel and root filesystem from the network
- U-Boot script
