Custom Image Builder for the Raspberry Pi 4 for ROS2 + PREEMPT_RT
=================================================================

This is a custom image builder for the Raspberry Pi 4. Some features:

- Customize the official Ubuntu server image for the Raspberry Pi by mounting
  it locally (via loop device) and chrooting into it (via systemd-nspawn and
  qemu-user-static).
  - I can't locate how Canonical generate the official Ubuntu images for the
    Raspberry Pi, so I had to resort to this method.
  - The default customization in this repo is made for ROS2 with `PREEMPT_RT`
    applied.
  - More information about how this works is below.
- With two stages of setup scripts, executing in lock step both inside and
  outside the chroot, we can cross compile code (via something like CMake
  toolchain) on the host and copy it into the chroot for making the final
  image.
- One thing we all hate while building images is to waste a lot of time. These
  scripts are designed to hopefully not waste your time. It has several
  features for this:
  - The build process is divided into resumable steps. This means there is no
    need (in most cases) to restart the build from scratch if you make a
    mistake. You can experiment with the image as it builds either by
    deliberately pausing the build process after a certain step or be forced to
    pause because there are some typos in the build scripts.
  - Nice logs that aids with debugging of the build, should things go wrong.
  - Use tools like `pv` to display progress when applicable.

How to use
----------

### System requirements

Unfortunately, the current setup doesn't work in Docker, as I used
`systemd-nspawn` to make setting up and executing commands in a chroot easier
(mainly so I can save some time figuring out the various bind mounts I need,
and to shutdown the container correctly if a command fails).  This tool also
rely on loop devices, which are not namespaced and thus not readily usable in
Docker without privileged access. Thus, you'll need a Linux machine with root
and the following tools installed: `cut`, `grep`, `parted`, `pv`, `rsync`,
`truncate`, `wget`, `systemd-nspawn`, and `qemu-aarch64-static`.

For Ubuntu, you can simply run:

```
$ sudo apt install parted pv rsync wget systemd-containers qemu-user-static make
```

### To run

```
$ make ros2-rt
```

This will build the image to `build/ubuntu-20.04.3-rt-ros2-galactic-arm64+raspi.img`.

You can see a demo of this in [CI](#TODO).

### Debugging your build

_This is covered in [How it works](#how-it-works)._

How it works
------------

_Note: the best way to understand the details is to start reading from
`builder/main.sh` and tracing every function call. Most functions are defined
in the order they are called in `builder/core.sh` and thus the code should read
somewhat linearly._

The builder defined in `builder/` is relatively generic and can theoretically
be used for a number of situations where a vendor provides you with a `.img`
file and you need to customize it. It follows these major steps:

1. Reads a custom `vars.sh` to populate all the variables needed to build.
2. Download the (ubuntu) image from the vendor.
3. Extract the image via a custom bash function defined in `vars.sh` to handle
   different compression algorithms.
4. Increase the size of the image and resize the file system, as the vendor
   image may be too small to install a large number of images.
5. Setup a loop device.
6. Mount the file systems in the loop device so they're accessible by the host.
7. Copy things like resolv.conf and qemu-user-static into the mounted FS.
8. Copy the rootfs overlay defined via `ROOTFS_OVERLAY` into the mounted FS.
  - If you have setup scripts running inside the chroot, or other files
    supporting the setup, put it here and copy it in.
9. Run the _host-side phase1 setup script_ (`setup_script_phase1_outside_chroot`).
  - For the RT setup, the RT kernel is built during this step.
  - You can also do things like downloading some prebuilt packages from the
    internet here.
10. Run the _chroot-side phase1 setup script_ (`setup_script_phase1_inside_chroot`).
    This runs the script inside the mounted FS via a chroot in the target
    architecture via qemu-user-static.
  - This is where you install things like the ROS2 packages.
11. Run the _host-side phase2 setup script_ (`setup_script_phase2_outside_chroot`).
  - If you want to cross compile, this is your chance, as the rootfs is readily
    available and mounted. In CMake, you can set your `CMAKE_FIND_ROOT_PATH` and
    `CMAKE_SYSROOT` to the path to the mounted FS.
12. Run the _chroot-side phase2 setup script_ (`setup_script_phase2_inside_chroot`).
    This also runs the script inside the mounted FS.
  - Usually this is when you remove any setup files you copied in the chroot
    and perform some cleanup. resolv.conf and other files setup by the builder
    will be cleaned by the builder, tho
13. Cleanup the chroot by removing resolv.conf and qemu-user-static.
14. Unmount everything and get rid of the loop back device.

To customize this process, look at the `vars.sh` file(s) in this repo. There
should be extensive comments there.

In the future, maybe it's better to figure out how Canonical generate their
official Ubuntu images. However, I can't find how they built their images when
I looked. It is likely that a number of steps here will be needed anyway, so
the existing structure is not that bad of an idea.

### How interrupt and resume works

Each step defined in the code is resumable (see `main.sh`). If they fail, the
builder won't clean everything up to give the developer a chance to debug and
possibly fix things manually before continuing. The workflow I take is as
follows:

1. Edit the build scripts. Run them.
2. Encounter an error.
3. Manually go into the chroot via systemd-nspawn.
4. Figure out the right commands to run and change the script.
5. Run the builder again and watch it resume from the failed step (can
   control to run or not to run the failed step).

This saves a lot of time, as you don't have to restart from the beginning when
encountering an error. The way this works is via two files saved in `cache`:

- `cache/session.txt`: contains a list of steps executed, one per line. You can
  freely change this file if you know what you're doing to selectively
  execute/skip steps when working with this system. See `builder/main.sh` for
  the steps (`run_step <step_name>`).
- `cache/session-loop-device.txt`: This saves the loop device the container is
  mounted to, since the loop devices may be different each time we run this.

There's also a `PAUSE_AFTER` variable that can be set in `vars.sh` to instruct
the builder to stop after a particular step. This allow you to do some
interactive experimentation, which also speeds things up.

To get into the container, run the command;

```
$ sudo systemd-nspawn TODO
```

### How to reset your host system if something horribly goes wrong

- Try running the commands in `umount_everything` manually (see
  `builder/core.sh`).
- If that doesn't work, try restarting your computer :(.

