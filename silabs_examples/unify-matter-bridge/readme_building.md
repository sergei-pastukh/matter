# Building the Unify Matter Bridge

This build guide cross-compiles for armhf architecture to be run on Unify's reference platform - a Raspberry Pi 4 (RPi4) with the 32-bit version of Debian Buster.

> **Note:**
> In the following subsections the commands should either be run on your local development machine or inside a running Docker container, as distinguished by the structure of the example.
> - _some-command_ should be executed on your local machine.
>   - _`dev-machine:~$ some-command`_ 
> - _some-other-command_ should be executed inside the Docker container.
>   - _`root@docker:/<dir>$ some-other-command`_


## Download and Stage the uic Repo

```bash
dev-machine:~$ git clone --depth 1 --branch external/matter-bridge-unstable https://github.com/SiliconLabs/UnifySDK.git --recursive ../uic-matter
```

## Build the Docker Container (armhf compilation)

```bash
dev-machine:~$ docker build -t unify-matter silabs_examples/unify-matter-bridge/docker/
```

Start the docker:

```bash
dev-machine:~$ docker run -it -v $PWD:/matter -v $PWD/../uic-matter:/uic unify-matter
```

## Build libunify

The Unify Matter Bridge depends on the libunify library from the Unify project.

This library must first be compiled for the target system, by changing directory to the `/uic` folder and running the following:

```bash
root@docker:/uic$ cmake -DCMAKE_INSTALL_PREFIX=$PWD/stage -GNinja -DCMAKE_TOOLCHAIN_FILE=../cmake/armhf_debian.cmake  -B build_unify_armhf/ -S components

root@docker:/uic$ cmake --build build_unify_armhf

root@docker:/uic$ cmake --install build_unify_armhf --prefix $PWD/stage
```

After cross-compiling the Unify library, set two paths in the PKG_CONFIG_PATH.
The first path is for the staged Unify library and the second is for cross compiling to armhf.

```bash
root@docker:/uic$ export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$PWD/stage/share/pkgconfig

root@docker:/uic$ export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/lib/arm-linux-gnueabihf/pkgconfig
```

## Set Up the Matter Build Environment

Once you have all the necessary submodules, source the Matter environment with the following command. This loads a number of build tools and makes sure the correct toolchains and compilers are used for compiling the Unify Matter Bridge.

## Check Out Submodules

Check out the necessary submodules with the following command.

```bash
dev-machine:~$ ./scripts/checkout_submodules.py --platform linux
```

```bash
root@docker:/matter$ source ./scripts/activate.sh
```

## Compile the Unify Bridge

```bash
root@docker:/matter$ cd silabs_examples/unify-matter-bridge/linux/

root@docker:/matter/silabs_examples/unify-matter-bridge/linux$ gn gen out/armhf --args='target_cpu="arm"'

root@docker:/matter/silabs_examples/unify-matter-bridge/linux$ ninja -C out/armhf
```

After building, the `unify-matter-bridge` binary is located at `/matter/silabs_examples/unify-matter-bridge/linux/out/armhf/obj/bin/unify-matter-bridge`.

## Compile the chip-tool

The `chip-tool` is a CLI tool that can be used to commission the bridge and to control end devices.

```bash
root@docker:/matter$ cd examples/chip-tool

root@docker:/matter/examples/chip-tool$ gn gen out/armhf --args='target_cpu="arm"'

root@docker:/matter/examples/chip-tool$ ninja -C out/armhf
```
After building, the chip-tool binary is located at `/matter/examples/chip-tool/out/armhf/obj/bin/chip-tool`.

## Unit Testing

Unit testing is always a good idea for quality software. Documentation on writing unit tests for the Matter Unify Bridge is in the
[README.md](linux/src/tests/README.md) in the `linux/src/tests` folder.

## Troubleshooting

1. If you do not source the `matter/scripts/activate.sh` as described above in [Set Up the Matter Build Environment](#set-up-the-matter-build-environment), `gn` and other common
   build tools will not be found.
2. If you do not export the `pkgconfig` for the `arm-linux-gnueabihf` toolchain as described above in [Build libunify](#build-libunify)
   you will get errors such as `G_STATIC_ASSERT(sizeof (unsigned long long) == sizeof (guint64));`
3. If you are compiling unit tests, do not try to compile the Unify Matter Bridge at
   the same time. This will not work as when compiling unit tests you are also
   compiling unit tests for all other sub-components.
4. If you encounter errors linking to `libunify`, try redoing the [`libunify` compile steps](#build-libunify).
5. Encountering problems with the submodules can be due to trying to check out
   the submodules inside the docker container.
