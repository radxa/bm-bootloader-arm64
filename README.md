# AIM BM1684X Ubuntu

Below is the instructions of how to build image for AIM BM1684X on a HOST PC.

## Download docker

Follow this guide, https://docs.docker.com/get-docker/, to install docker first on HOST PC.

Download radxadev/bm1684x docker.

```
docker pull radxadev/bm1684x:v1.0
```

Start container

radxadev-bm1684x-v1.0 is just an example, you can set your own name.

```
docker run --rm --privileged --name radxadev-bm1684x-v1.0 -v $PWD:/workspace -it radxadev/bm1684x:v1.0
```

## Get the source code

```
$ export AIM_BM1684X_SDK="/workspace/aim-bm1684x-sdk"
$ mkdir -p ${AIM_BM1684X_SDK}/distro ${AIM_BM1684X_SDK}/prebuilt

$ cd ${AIM_BM1684X_SDK}
$ git clone https://github.com/radxa/bm-bootloader-arm64.git -b aim-bm1684x bootloader-arm64
$ git clone https://github.com/radxa/bm-linux-arm64.git -b aim-bm1684x linux-arm64
$ git clone https://github.com/radxa/libsophon.git -b aim-bm1684x

$ cd ${AIM_BM1684X_SDK}/prebuilt
$ wget https://github.com/radxa/bootloader-arm64/releases/download/20231212/distro_focal.tgz
$ wget https://github.com/radxa/bootloader-arm64/releases/download/20231212/linaro.tgz
$ wget https://github.com/radxa/bootloader-arm64/releases/download/20231212/sophon-mw-soc-sophon-ffmpeg_0.7.1_arm64.deb
$ wget https://github.com/radxa/bootloader-arm64/releases/download/20231212/sophon-mw-soc-sophon-opencv_0.7.1_arm64.deb

$ cp ${AIM_BM1684X_SDK}/prebuilt/distro_focal.tgz ${AIM_BM1684X_SDK}/distro
$ tar zxvf ${AIM_BM1684X_SDK}/prebuilt/linaro.tgz -C ${AIM_BM1684X_SDK}
```

## Start building

### Step one: Build bootloader and kernel

```
$ export AIM_BM1684X_SDK="/workspace/aim-bm1684x-sdk"
$ export PATH="${AIM_BM1684X_SDK}/gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu/bin:$PATH"
$ cd ${AIM_BM1684X_SDK}
$ source bootloader-arm64/scripts/envsetup.sh
$ build_bsp_without_package
```

### Step two: Build libsophon

```
$ mkdir -p ${AIM_BM1684X_SDK}/libsophon/build/soc_kernel

$ LINUX_HEADER=$(basename `find ${AIM_BM1684X_SDK}/install/soc_bm1684/bsp-debs/linux-headers-5.4.217-bm1684*.deb` .deb)

$ dpkg -X ${AIM_BM1684X_SDK}/install/soc_bm1684/bsp-debs/${LINUX_HEADER}.deb ${AIM_BM1684X_SDK}/libsophon/build/soc_kernel
$ cd ${AIM_BM1684X_SDK}/libsophon/build
$ cmake -DPLATFORM=soc -DSOC_LINUX_DIR=${AIM_BM1684X_SDK}/libsophon/build/soc_kernel/usr/src/${LINUX_HEADER}/ -DLIB_DIR=${AIM_BM1684X_SDK}/libsophon/3rdparty/soc/ \
      -DCROSS_COMPILE_PATH=${AIM_BM1684X_SDK}/gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu \
      -DCMAKE_TOOLCHAIN_FILE=${AIM_BM1684X_SDK}/libsophon/toolchain-aarch64-linux.cmake \
      -DCMAKE_INSTALL_PREFIX=$PWD ..
$ make
$ make driver
$ make vpu_driver
$ make jpu_driver
$ make package
```

### Step three: Build final images

Copy ffmpeg and opencv pacakges to `install/soc_bm1684/bsp-debs` directory.
Copy libsophon packages to `install/soc_bm1684/bsp-debs` directory.

```
$ cp ${AIM_BM1684X_SDK}/prebuilt/sophon-mw-soc-sophon-ffmpeg_0.7.1_arm64.deb ${AIM_BM1684X_SDK}/install/soc_bm1684/bsp-debs
$ cp ${AIM_BM1684X_SDK}/prebuilt/sophon-mw-soc-sophon-opencv_0.7.1_arm64.deb ${AIM_BM1684X_SDK}/install/soc_bm1684/bsp-debs

$ cp ${AIM_BM1684X_SDK}/libsophon/build/sophon-soc-libsophon*.deb ${AIM_BM1684X_SDK}/install/soc_bm1684/bsp-debs
```

Run `build_pacakge`

```
$ cd ${AIM_BM1684X_SDK}
$ build_package
```

And we will get final system images under `install/soc_bm1684/sdcard/`.

## Write System image

BM1684X boots from SPI Nor Flash and eMMC.
Here we use Micro SD card to write images to SPI Nor Flash and eMMC.

### Create one Micro SD card with system image and tools

#### Step one: Format Micro SD card

We need to format Micro SD card. Please make sure that the disklabel is dos.
And the first partition of the Micro SD card is FAT32 with a size of 1GB or more.

#### Step two: Put image files to Micro SD card

Copy all the files under directory `install/soc_bm1684/sdcard/` to the first partition of the Micro SD card.

### Start writing images

#### Step one: Connect the serial debug cable to check the log.

For example, Radxa Fogwise BM1684X is equipped with Type-C debug port. You can use one Type-A to Type-C cable to connect it to your PC.
The baudrate is 115200.

#### Step two: Insert Micro SD card to BM1684X board.

#### Step three: Power on BM1684X board

Power on BM1684X board and the writing process will automatically start. Once done, the system shows removing media device.

#### Step four: Power cycle BM1684X board.

