# AIFM
![Status](https://img.shields.io/badge/Version-Experimental-green.svg)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

AIFM stands for Application-Integrated Far Memory. It provides a simple, general, and high-performance mechanism for users to adapt ordinary memory-intensive applications to far memory. Different from existing paging-based systems, AIFM exposes far memory as far-memory pointers and containers in the language level. AIFM's API allows its runtime to accurately capture application semantics, therefore making intelligent decisions on data placement and movement.

Currently, AIFM supports C++ and TCP-enabled remote server memory.

- [AIFM](#aifm)
  * [Paper](#paper)
  * [Supported Platform](#supported-platform)
  * [Build Instructions](#build-instructions)
    + [Install Dependencies (on all nodes)](#install-dependencies-on-all-nodes)
    + [Build Shenango and AIFM (on all nodes)](#build-shenango-and-aifm-on-all-nodes)
    + [Setup Shenango (on all nodes)](#setup-shenango-on-all-nodes)
    + [Configure AIFM (only on the compute node)](#configure-aifm-only-on-the-compute-node)
  * [Run AIFM Tests](#run-aifm-tests)
  * [Reproduce Experiment Results](#reproduce-experiment-results)
  * [Repo Structure](#repo-structure)
  * [Known Limitations](#known-limitations)
  * [Contact](#contact)

## Paper
* [AIFM: High-Performance, Application-Integrated Far Memory](https://www.usenix.org/conference/osdi20/presentation/ruan)<br>
Zhenyuan Ruan, Malte Schwarzkopf, Marcos Aguilera, Adam Belay<br>
The 14th USENIX Symposium on Operating Systems Design and Implementation (OSDI ‘20)

## Supported Platform

Ubuntu 20.04, linux 5.4.0, gcc 9.4.0

## Build Instructions

### Install Dependencies (on all nodes)
You have to install the necessary dependencies in order to build AIFM. Note you have to do run those steps on all nodes.

1) Enable monitor/mwait.

Enable monitor/mwait in BIOS.

Reboot and check it with
```
lscpu | grep monitor
```

2) Enable intel idle driver.

Make sure `idle=poll/hlt/nomwait` is not set in kernel cmdline in file `/etc/default/grub`.

Check current driver with
```
cat /sys/devices/system/cpu/cpuidle/current_driver
```
It should be `intel_idle`. The `none` result means that the cpu is not supported by intel idle driver. However you could modify linux 5.4.0
source code to make current cpu recognized by the driver.

Fisrt check the cpu mode with
```
dmesg | grep intel_idle
```
Suppose the result is `intel_idle: does not run on family 6 model 106` where **106 (0x6A)** is cpu mode and
**ICELAKE X** is microarchitecture codename. Then add related code to `linux/arch/x86/include/asm/intel-family.h` and
`linux/drivers/idle/intel_idle.c` by referring to **SKYLAKE X**.

3) Install Mellanox OFED.
```
wget "https://content.mellanox.com/ofed/MLNX_OFED-4.9-5.1.0.0/MLNX_OFED_LINUX-4.9-5.1.0.0-ubuntu20.04-x86_64.iso"
sudo mount -o ro,loop MLNX_OFED_LINUX-4.9-5.1.0.0-ubuntu20.04-x86_64.iso /mnt
sudo /mnt/mlnxofedinstall --add-kernel-support --dpdk --upstream-libs # it's fine to see 'Failed to install libibverbs-dev DEB'
sudo /etc/init.d/openibd restart
```

4) Configure IB NIC.

Switch mlx5 nic port to eth mode
```
sudo mstconfig -d <pci addr> set LINK_TYPE_P<port id>=2
sudo mstfwreset -d <pci addr> -l3 -y reset
```

5) Install libraries and tools.
```
echo Y | sudo apt-get --fix-broken install
echo Y | sudo apt-get install libnuma-dev libmnl-dev libnl-3-dev libnl-route-3-dev
echo Y | sudo apt-get install libcrypto++-dev libcrypto++-doc libcrypto++-utils
echo Y | sudo apt-get install software-properties-common
echo Y | sudo apt-get install gcc-9 g++-9 python-pip
echo Y | sudo add-apt-repository ppa:ubuntu-toolchain-r/test
echo Y | sudo apt-get purge cmake
sudo pip install cmake
```
6) Set bash as the default shell.
```
chsh -s /bin/bash
```

### Build Shenango and AIFM (on all nodes)
For all nodes, clone our github repo in a same path, say, your home directory. 

AIFM relies on Shenango's threading and TCP runtime.

The ib device name should be hard coded with `strncmp(ibv_get_device_name(ib_dev), "mlx5", 4)` in
file `runtime/net/directpath/mlx5/mlx5_init.c`.

The ib device port should also be hard coded with `dp.port = 0` in file
`shenango/iokernel/dpdk.c`.

The `build_all.sh` script in repo root compiles both Shenango and AIFM automatically.
```
./build_all.sh
```

### Setup Shenango (on all nodes)
After rebooting machines, you have to rerun the script to setup Shenango.
```
sudo ./scripts/setup_machine.sh
```

### Configure AIFM (only on the compute node)
So far you have built AIFM on both nodes. One node is used as the compute node to run applications, while the other node is used as the remote memory node. Now edit `aifm/configs/ssh` in the compute node; change `MEM_SERVER_SSH_IP` to the IP of the remote memory node (eno49 inet in `ifconfig`), and `MEM_SERVER_SSH_USER` to your ssh username. Please make sure the compute node can ssh the remote memory node successfully without password.

## Run AIFM Tests
Now you are able to run AIFM programs. `aifm/test` contains a bunch of test files of using local/far pointers and containers (and few other system components). You can also treat those tests as examples of using AIFM. `aifm/test.sh` is a script that runs all tests automatically. It includes the commands of running AIFM end-to-end.
```
./test.sh
```

## Reproduce Experiment Results
We provide code and scripts in `aifm/exp` folder for reproducing our experiments. For more details, see `aifm/exp/README.md`.

## Repo Structure

```
Github Repo Root
 |---- build_all.sh  # A push-button build script for building both Shenango and AIFM.
 |---- shenango      # A modified version of Shenango runtime for AIFM. DO NOT USE OTHER VERSIONS.
 |---- aifm          # AIFM code base.
        |---- bin       # Test binaries and a TCP server ran at the remote memory node.
        |---- configs   # Configuration files for running AIFM.
        |---- inc       # AIFM headers.
        |---- src       # AIFM cpp files.
        |---- test      # Test files of using far-memory pointers and containers.
        |---- snappy    # An AIFM-enhanced snappy.
        |---- DataFrame # C++ DataFrame library (which includes both the original version and the AIFM version).
        |---- exp       # Code and scripts for reproducing our experiments.
        |---- Makefile
        |---- build.sh  # The script for building AIFM.
        |---- test.sh   # The script for testing AIFM.
        |---- shared.sh # A collection of helper functions for other scripts.
```

## Known Limitations
AIFM is a research prototype rather than a production-ready system. Its current implementation has two main limitations.
1. A thread cannot have more than one live `DerefScope` at any time (but you can have multiple live `DerefScope`s across different threads). For example, when executing `foo()->bar()` in a thread, if you've already instantiated a `DerefScope` in `foo()`, you must not instantiate another one in `bar()`. The right way is to pass the one in `foo()` as a reference to `bar()`.
2. AIFM assumes that the __remote__ memory is sufficiently large so that it never garbage collects the dead (i.e., freed) objects in the __remote__ memory, see `FarMemManager::mutator_wait_for_gc_far_mem()` in `aifm/src/manager.cpp`. However, AIFM does garbage collects the dead objects in the __local__ memory since it's a precious resource.

## Contact
Contact zainruan [at] csail [dot] mit [dot] edu for assistance.
