## So You Want To Make A Nethunter Kernel

####Introduction
The biggest hurdle for most people who want to set up a Nethunter device is the kernel requirement. Unless your device already has a prebuilt (and maintained) kernel, then you will be told to build one yourself. There is a lot of documentation on this process already, but in my experience it can be difficult to know what applies in your case and what does not.
This problem is confounded by ~~assholes~~ people who write guides with clickbait-y titles like“Build a Nethunter kernel for ANY Android device!”or with false promises like“Learn how to compile a Nethunter kernel in ten minutes!”It is also an unfortunate reality of the internet in 2025 that many ~~idiots~~ people think that ChatGPT or other large language models have authoritative answers to their questions, when all that those models really have is information they scraped from the aforementioned assholes.
I am writing this guide as thoroughly and honestly as I possibly can. It is **not** intended to be the **only **guide you will need for this process, however – surely you will have questions that this document cannot answer. I would like to encourage you to search for answers to these questions **in the relevant software documentation**, and **NOT** on YouTube or from ChatGPT.

####Requirements
Before you begin ANY of this process, you should have ALREADY:

- Unlocked the bootloader on your device
- Installed an AOSP (Android Open-Source Project) based ROM (LineageOS, crDroid, etc)
*- (optional: installed a custom recovery)*
- Rooted your phone with Magisk 28.1 + enabled Zygisk
*- (if you have a newer version of Magisk, reboot to recovery and flash Magisk 28.1)*
- Flashed the kali-nethunter-(year.release-number)-generic-arm64-minimal.zip in Magisk
- Check that the Nethunter app + Nethunter terminal app work
- Updated all installed packages `apt update && apt upgrade -y`
*- (optional: added metapackages)*
- Flashed the wireless firmware for Nethunter Magisk module

**In order to complete this process, you will need:**

- A desktop/laptop computer using x86_64 processor architecture (Intel/AMD) with GNU/Linux installed
*- (yes, you can use a virtual machine or a Docker instance if necessary, but I cannot orient you on the usage of such things, as I simply run GNU/Linux on “bare metal” and would recommend you do the same for performance reasons)*
- At least 8GB of RAM + 20GB of free disk space
- A good internet connection
*- (semi-optional: an account on GitHub or GitLab. This will make it easier for you to track your changes, and also to distribute your kernel if you are successful in building it. If you do not make an account, then do not expect to share your kernel with anyone else – nobody is interested in a non-GPL compliant kernel image that does not come with source code!)*
- Basic knowledge of navigating the Linux command line (`cd, rm, ls, mkdir, find, diff, grep,` etc)
- Basic knowledge of using a command-line based editor (vim for the cool kids, otherwise nano)
- Basic knowledge of C programming language/Makefile syntax
- Your distribution’s build-from-source metapackage installed (`build-devel` on Arch-based distros, `build-essential` on Debian based) – I would also recommend you install Perl + Python3 as some kernel makefiles require these

**Is this going to be possible for my device?**

The Linux kernel is licensed under the GNU General Public License 2.0, which is a copyleft license. This means that the source code for the Linux kernel is freely distributed and you may alter the source code as you wish (as each Android manufacturer has done), HOWEVER, your resulting source code must ALSO be distributed freely. 
To the surprise of absolutely no one, many of the gigantic corporations who use GPL-licensed code flaunt the terms of this license. They either refuse to distribute their source code or only release part of it (usually broken, outdated, and with inline binary blobs). **If there is no source code available for the kernel for your device, then there is no way to create a Nethunter kernel for it.**
Your ability to unlock the bootloader and install a custom ROM also depends greatly on the whims of your device manufacturer. Even if they allow unlocking the bootloader and you have been able to root your phone with Magisk, if they do not publish device trees, there are likely no custom ROMs for your device. In this case, it is extremely doubtful that the kernel source code they published (if any) has received any maintenance since its release, or that it will build even without modifications. **If you do not see anyone else maintaining anything (kernels, ROMs, recoveries) for your device, then it will be difficult, if not impossible for you to turn that device into a full Nethunter. **

**A Note On Kernel Versions**

Let us imagine that there are regularly updated and maintained custom ROMs/kernels/recoveries for your device, but simply no Nethunter kernel. In this case, you should be able to complete this process, but the process will vary depending on which version of the Linux kernel your device uses. 
The information in this guide is based on my experience modifying 4.14.x kernels. There are older guides written that talk about using GCC toolchains (instead of LLVM/clang, as we will be using) that apply to 3.x kernels. **If your device uses a 5.x/6.x version of the Linux kernel, then the information in this guide will not be sufficient to complete this process.** The Android build method changes all the time, and you should look for a guide that covers GKI kernels and the use of Bazel/Kleaf.

####First Steps: Source, Toolchain, and Configuration


A good way to find repos containing the kernel source for your device is to use its codename. Every Android has one, although some manufacturers (i.e. OnePlus) have a bit more fun with this than others (i.e. Samsung). The codename for my Xiaomi Poco X3 NFC is "surya", whereas the codename for my Samsung A51 is "SM-A515F". Typically, kernel source repos follow the naming convention
` android_kernel_MANUFACTURER_CODENAME `
where the manufacturer is the parent company, i.e. `android_kernel_xiaomi_surya`, not `android_kernel_poco_surya`. 
However, there may also be repos based around the SoC (system-on-a-chip) for your device, sometimes “unified” to include source files for other devices with the same SoC. This is the case for the Samsung I mentioned, whose source repo is `called android_kernel_samsung_exynos9611`. Have a look through github/gitlab and see what you can find.

For the purposes of this guide, I will be using the latest LineageOS 22.2 (built 2025-08-04) on surya, therefore, I will want to clone the kernel source from the LineageOS repo. But before I do that, I want to copy two files from my device to the computer I will be using to build the kernel: `/proc/config.gz` and `/proc/version`.
I could copy the first file, `/proc/config.gz`, to `/sdcard`, pull it from the device with ADB, gunzip it, and rename it. But that seems like a lot of steps, so what I did instead was this:
