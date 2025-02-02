# MLPerf Inference v4.0 NVIDIA-Optimized Inference on Jetson Systems
This is a repository of NVIDIA-optimized implementations for the [MLPerf](https://mlcommons.org/en/) Inference Benchmark.
This README is a quickstart tutorial on how to setup the the Jetson systems as a public / external user.
Please also read README.md for general instructions on how to run the code.

---

NVIDIA Jetson is a platform for AI at the edge. Its high-performance, low-power computing for deep learning and computer vision makes it the ideal platform for compute-intensive projects. The Jetson platform includes a variety of Jetson modules together with NVIDIA JetPack™ SDK.
Each Jetson module is a computing system packaged as a plug-in unit (a System on Module (SOM)). NVIDIA offers a variety of Jetson modules with different capabilities.
JetPack bundles all of the Jetson platform software, starting with NVIDIA Jetson Linux. Jetson Linux provides the Linux kernel, bootloader, NVIDIA drivers, flashing utilities, sample file system, and more for the Jetson platform.

## NVIDIA Submissions

The Jetson AGX Orin submission supports:

- Stable Diffusion XL (Offline, Single Stream)
- GPT-J (Offline and Single Stream), at 99% and 99.9% of FP32 accuracy target

*Note: power submission is not supported in MLPerf Inference v4.0*

## Setup the Jetson AGX Orin System

### Flash the board
Follow the the [Jetson Developer Guide](https://docs.nvidia.com/jetson/archives/r35.3.1/DeveloperGuide/text/IN/QuickStart.html#to-flash-the-jetson-developer-kit-operating-software) to flash the board with the r35.3.1 L4T

For replicating performance submission on AGX dev kit, flash the AGX board with **MaxN config**, e.g. `sudo ./flash.sh jetson-agx-orin-devkit-maxn mmcblk0p1`

### Apply Custom 64k Page Size Kernel
NVIDIA's Orin submission systems uses a custom 64k Page size kernel on all systems.

After flashing the Jetson board, please follow the steps below to build apply a 64K page size custom kernel. All steps should be executed on the board instead of the host.

#### Step 1: Download and prepare the kernel source
Make sure you are on the target system (the board), and download the kernel source from [Jetson Linux Archive](https://developer.nvidia.com/embedded/jetson-linux-r3531). Choose Driver Package (BSP) Sources to download

Extract the kernel source by running
```
tar -xjf public_sources.tbz2
cd Linux_for_Tegra/source/public
tar –xjf kernel_src.tbz2
```

#### Step 2: Install software dependencies
Install necessary software to flash the board with
```
sudo apt-get install make build-essential bc libncurses-dev bison flex libssl-dev libelf-dev
```

#### Step 3: Enable 64k page size config and build the kernel
Modifiy the config file under `/Linux_for_Tegra/source/public/kernel/kernel-5.10/arch/arm64/configs/tegra_defconfig` by adding the following flag
```
CONFIG_ARM64_64K_PAGES=y
```

Compile the kernel
```
mkdir kernel_out
./nvbuild.sh -o $PWD/kernel_out
```

#### Step 4: Build the L4T modules
Enter the kernel build directory and build the modules
```
cd kernel_out
sudo make -j8 INSTALL_MOD_PATH=<some-path-to-store-your-module-build> O= modules_install
```

#### Step 5: Apply the image and modules
Apply the image to the system
```
sudo cp -v arch/arm64/boot/Image /boot/Image
```

Apply the modules to the system
```
cd <some-path-to-store-your-module-build>/lib/modules/5.10.104-tegra
sudo cp -rv kernel /lib/modules/5.10.104-tegra/
sudo cp -rv module* /lib/modules/5.10.104-tegra/
```

#### Step 6: Reboot
```
sudo reboot
```

Wait for a few minutes and you should be able to access the system again. You can verify that 64k page size kernel has taken affect with
```
getconf PAGESIZE
65536
```

#### Tips
The built kernel Image and the modules and be reused to patch the system again after reflashing. Please make sure those files' permission is root:root.

#### Alternative cross compile option
For user interested in cross compiling the kernel, please check [kernel customization](https://docs.nvidia.com/jetson/archives/r34.1/DeveloperGuide/text/SD/Kernel/KernelCustomization.html#building-the-kernel) and ask NVIDIA L4T team for any questions.


### Troubleshooting

- System Non-operational After Flashing the L4T Image`

Make sure the following dependencies are installed on your host system

```
sudo apt update && sudo apt install -y libxml2-utils & sudo apt install -y qemu-user-static
```

- No GPU detected after applying the custom kernel

Make sure you have the modules built and replaced the system modules following the instructions in Step5 above.


## Apply Perf Optmization for AGX

### Lock the max clocks

The best ML performance can be achieved with highest power settings. In NVIDIA published Orin benchmarks, the board were configured to use MAXN power profile. The MAXN mode unlock GPU/DLA/CPU/EMC TDP restrictions and is available through flash config switch.

Run the following command under open/NVIDIA/ to lock the max clocks
```
sudo jetson_clocks
sudo jetson_clocks --fan
make apply_jetson_oc_limit
```

### USB-C Power Adapters

For performance submission systems, Dell 130.0W Adapter (HA130PM170) were used for Jetson AGX.

## Download Data, Model and Preprocess the data

The data downloading, model downloading and data preprocessing have to be done on **a x86 system**. Please follow the section **Setting up the Scratch Spaces**, **Download the Datasets**, **Downloading the Model files**, **Preprocessing the datasets for inferences** described in [README.md](README.md) to perform the procedure.

## Running a Benchmark

As noted in [README.md](README.md) all benchmarks need to run inside a docker container. Follow the steps in **Running your first benchmark** in the main [README.md](README.md) for instructions on running the benchmarks.

## FAQ
- Is it a must to apply the 64k page kernel?

To replocate NVIDIA's Jetson AGX Orin submission results, it is necessary to have the 64k page kernel.


- I encountered error when launching the docker. The code throws error `Error response from daemon: could not select device driver "" with capabilities: [[gpu]].
nvidia docker is not installed`

Please make sure you install [NVIDIA container Toolkit](https://github.com/NVIDIA/nvidia-docker) and set it as the default container runtime. You can also set the container runtime by adding `--runtime=nvidia` to the `DOCKER_ARGS`. E.g. `DOCKER_ARGS="--runtime=nvidia"`

- I encountered cpu permission error when launching the docker. The code throws error `ocker: Error response from daemon: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: unable to apply cgroup configuration: failed to write "0-11"`

Please check the value of `/sys/fs/cgroup/cpuset/docker/cpuset.cpus` and `/sys/fs/cgroup/cpuset/cpuset.cpus`. You need to reset the value of `/sys/fs/cgroup/cpuset/docker/cpuset.cpus` with the value of `/sys/fs/cgroup/cpuset/cpuset.cpus`. E.g. `sudo echo "0-11" > /sys/fs/cgroup/cpuset/docker/cpuset.cpus`
