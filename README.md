## Instructions for Getting DPDK Drivers Up and Running on AMD OpenNIC 

This is one of the three components of the
[OpenNIC project](https://github.com/Xilinx/open-nic.git).  The other components are:
- [OpenNIC shell](https://github.com/Xilinx/open-nic-shell.git) and
- [OpenNIC driver](https://github.com/Xilinx/open-nic-driver.git).

This OpenNIC DPDK repo contains a series of patch files and instructions with details for building DPDK with drivers for [OpenNIC](https://github.com/Xilinx/open-nic).  The basic sections are:

1. Install Build Dependencies.
1. The drivers at [Xilinx QDMA's DPDK driver repo](https://github.com/Xilinx/dma_ip_drivers) need to be patched for OpenNIC using the patch files contained in this repo.
1. Download DPDK and DPDK pktgen. The DPDK 23.11 distribution needs minor edits to include these drivers.
1. Build.
1. Configure proc/cmdline and BIOS if necessary.
1. Run Vivado to generate the [open-nic-shell](https://github.com/Xilinx/open-nic-shell) configured for two CMAC ports and two PFs for testing these.
1. Gives examples for writing to the hardware registers in order to set up the test.
1. Gives examples for binding DPDK to devices and testing this driver by running pktgen for the two PFs.

The rest of this document contains the step-by-step instructions for each of the sections listed above.  These instructions were written assuming Ubuntu e.g. 18.04 or 20.04.

### Section 1: Install Build Dependencies

1.  Install dependencies for building DPDK:
    ```
    sudo apt install build-essential
    sudo apt install libnuma-dev
    sudo apt install pkg-config
    sudo apt install python3 python3-pip python3-setuptools
    sudo apt install python3-wheel python3-pyelftools
    sudo apt install ninja-build
    sudo pip3 install meson
    ```    

1.  Install dependencies for building pktgen-dpdk:
    ```
    sudo apt install libpcap-dev
    ```
    
    Substitute the kernel version from "uname -a" into the command
    below:
    ```
    sudo apt install linux-headers-5.4.0-96-generic
    ```


### Section 2: Download Xilinx QDMA DPDK driver and Apply OpenNIC Patches

1.  This repo contains several patch files that must be applied to the [Xilinx QDMA DPDK drivers](https://github.com/Xilinx/dma_ip_drivers.git).
    ```
    git clone https://github.com/Xilinx/dma_ip_drivers.git
    cd dma_ip_drivers
    git checkout 87c54f5
    cd ..
    ```

1.  Clone this open-nic-dpdk repo:
    ```
    git clone https://github.com/Xilinx/open-nic-dpdk
    ```

1.  Copy the `*.patch` files contained in this repo into the QDMA driver's directory.
    ```
    cp open-nic-dpdk/*.patch dma_ip_drivers
    ```
    
1.  Then apply the OpenNIC patches:
    ```
    cd dma_ip_drivers
    git apply *.patch
    cd ..
    ```

### Section 3: Download DPDK and pktgen-dpdk

1.  Download [DPDK source](https://fast.dpdk.org/rel/dpdk-23.11.tar.xz) and build it, including the QDMA drivers.

    ```
    wget https://fast.dpdk.org/rel/dpdk-23.11.tar.xz
    tar xvf dpdk-23.11.tar.xz
    cd dpdk-23.11
    cp -R ../dma_ip_drivers/QDMA/DPDK/drivers/net/qdma ./drivers/net
    cp -R ../dma_ip_drivers/QDMA/DPDK/examples/qdma_testapp ./examples
    ```

2.  Additionally, make below changes to the DPDK 23.11 tree to build QDMA driver,
	support 4K queues and populate Xilinx devices for binding.

		i. Add QDMA driver
			a. To support 4K queues and 256 PCIe functions, update below configurations	in ./config/rte_config.h
                #define RTE_MAX_QUEUES_PER_PORT 4096

			b. Update below lines to ./config/meson.build in DPDK 23.11 tree
				# Set maximum Ethernet ports to 256
                from: 
                    dpdk_conf.set('RTE_MAX_ETHPORTS', get_option('max_ethports'))
                to: 
				    dpdk_conf.set('RTE_MAX_ETHPORTS', 256)

				# Set maximum VFIO Groups to 256
                from:
                    dpdk_conf.set('RTE_MAX_VFIO_GROUPS', 64)
                to:
				    dpdk_conf.set('RTE_MAX_VFIO_GROUPS', 256)

			c. Add below lines to ./config/rte_config.h to enable driver debug logs
				#define RTE_LIBRTE_QDMA_DEBUG_DRIVER 1

			d. Add below line to ./drivers/net/meson.build, where PMDs are added to drivers list
				'qdma',

			e. To add Xilinx devices for device binding, add below lines to	./usertools/dpdk-devbind.py after cavium_pkx class, where PCI base class for devices are listed.
				xilinx_qdma_pf = {'Class':  '05', 'Vendor': '10ee', 'Device': '9011,9111,9211,9311,9014,9114,9214,9314,9018,9118,9218,9318,901f,911f,921f,931f,9021,9121,9221,9321,9024,9124,9224,9324,9028,9128,9228,9328,902f,912f,922f,932f,9031,9131,9231,9331,9034,9134,9234,9334,9038,9138,9238,9338,903f,913f,923f,933f,9041,9141,9241,9341,9044,9144,9244,9344,9048,9148,9248,9348,b011,b111,b211,b311,b014,b114,b214,b314,b018,b118,b218,b318,b01f,b11f,b21f,b31f,b021,b121,b221,b321,b024,b124,b224,b324,b028,b128,b228,b328,b02f,b12f,b22f,b32f,b031,b131,b231,b331,b034,b134,b234,b334,b038,b138,b238,b338,b03f,b13f,b23f,b33f,b041,b141,b241,b341,b044,b144,b244,b344,b048,b148,b248,b348', 'SVendor': None, 'SDevice': None}
				xilinx_qdma_vf = {'Class':  '05', 'Vendor': '10ee', 'Device': 'a011,a111,a211,a311,a014,a114,a214,a314,a018,a118,a218,a318,a01f,a11f,a21f,a31f,a021,a121,a221,a321,a024,a124,a224,a324,a028,a128,a228,a328,a02f,a12f,a22f,a32f,a031,a131,a231,a331,a034,a134,a234,a334,a038,a138,a238,a338,a03f,a13f,a23f,a33f,a041,a141,a241,a341,a044,a144,a244,a344,a048,a148,a248,a348,c011,c111,c211,c311,c014,c114,c214,c314,c018,c118,c218,c318,c01f,c11f,c21f,c31f,c021,c121,c221,c321,c024,c124,c224,c324,c028,c128,c228,c328,c02f,c12f,c22f,c32f,c031,c131,c231,c331,c034,c134,c234,c334,c038,c138,c238,c338,c03f,c13f,c23f,c33f,c041,c141,c241,c341,c044,c144,c244,c344,c048,c148,c248,c348', 'SVendor': None, 'SDevice': None}

			d. Update entries in network devices class in ./usertools/dpdk-devbind.py to add Xilinx devices
				network_devices = [network_class, cavium_pkx, avp_vnic, ifpga_class, xilinx_qdma_pf, xilinx_qdma_vf]

3.  Return to the earlier directory:
    ```
    cd ..
    ```

4.  Download [pktgen-dpdk source](https://git.dpdk.org/apps/pktgen-dpdk/snapshot/pktgen-dpdk-pktgen-23.10.0.tar.xz) and build it, including the QDMA drivers.
    ```
    wget \
    https://git.dpdk.org/apps/pktgen-dpdk/snapshot/pktgen-dpdk-pktgen-23.10.0.tar.xz
    tar xvf pktgen-dpdk-pktgen-23.10.0.tar.xz
    ```

### Section 4: Build



1.  Build DPDK:
    ```
    cd dpdk-23.11
    meson build
    cd build
    ninja
    sudo ninja install
    ls -l /usr/local/lib/x86_64-linux-gnu/librte_net_qdma.so
    sudo ldconfig
    ls -l ./app/test/dpdk-test
    cd ../..
    ```

1.  Build pktgen-dpdk:
    ```
    cd pktgen-23.10.0
    make RTE_SDK=../dpdk-23.11 RTE_TARGET=build
    ```


### Section 5: Configure proc/cmdline and BIOS if necessary


1.  Make sure that IOMMU is enabled within the BIOS settings.

    *Note: Enable VT-d for Intel processors within the BIOS.*

1.  Set grub settings to enable hugepages and IOMMU if necessary. The
    following example grub command line below is based on an AMD machine
    with e.g. 16GB of RAM, so please adjust the number of hugepages below
    as appropriate.

    Edit `/etc/default/grub` to include the following line:

    `GRUB_CMDLINE_LINUX=" default_hugepagesz=1G hugepagesz=1G hugepages=4"`

    *Note: Add* `intel_iommu=on` *above for Intel processors.*

1.  Update grub:
    ```
    sudo update-grub
    ```

1.  Reboot for the changes to take effect.
    ```
    sudo reboot
    ```

1.  Confirm that hugepages appears within the `/proc/cmdline`:
    ```
    cat /proc/cmdline
    ```
    
### Section 6: Create the OpenNIC bitfile configured for two CMAC ports

1.  Clone the latest version of the open-nic-shell from github (including
    some updates after 1.0 release).
    ```
    git clone https://github.com/Xilinx/open-nic-shell.git
    ```
1.  Follow the instructions within
    <https://github.com/Xilinx/open-nic-shell> for building the bitfile
    including specifying the appropriate board and also the following
    parameters: `-num_cmac_port 2 -num_phys_func 2`.

1.  Open Vivado to complete the implementation and to generate the bitfile.

1.  Use Vivado to load the bitfile.

1.  Either reboot or pci rescan to redectect the pci devices, for example:
    ```
    echo 1 | sudo tee /sys/bus/pci/devices/0000\:d7\:00.0/rescan
    sudo setpci -s d8:00.0 COMMAND=0x02
    ```

### Section 7: Writing to Initialize the OpenNIC hardware registers

1.  Find the pcie bus and device ID for the two PFs:
    ```
    $ lspci -d 10ee:
    ```
    > 08:00.0 Memory controller: Xilinx Corporation Device 903f
    >
    > 08:00.1 Memory controller: Xilinx Corporation Device 913f

    Below is an example of finding the sysfile name of the device ID:
    ```
    $ lspci -td 10ee:
    ```
    > -+-[0000:d8]-+-00.0
    > |           \-00.1
    > \-[0000:00]-
    ```
    $ cd -P /sys/bus/pci/devices/0000:d8:00.0 && pwd
    ```
    > /sys/devices/pci0000:d7/0000:d7:00.0/0000:d8:00.0

1.  Use a utility program such as pcimem or similar to write to enable
        the CMAC registers and the QDMA.

    The following example writes should be modified for your pcie device
    sysfile name. Perform the following register writes.
    
    Enable the PCIe device for writing:
    ```
    sudo setpci -s 08:00.0 COMMAND=0x02;

    sudo setpci -s 08:00.1 COMMAND=0x02;
    ```
    Write to QDMA:
    ```
    sudo pcimem \
    /sys/devices/pci0000:00/0000:00:03.1/0000:08:00.0/resource2 0x1000 w 0x1;

    sudo pcimem \
    /sys/devices/pci0000:00/0000:00:03.1/0000:08:00.0/resource2 0x2000 w 0x00010001;
    ```
    Write to enable CMAC0:
    ```
    sudo pcimem \
    /sys/devices/pci0000:00/0000:00:03.1/0000:08:00.0/resource2 0x8014 w 0x1;

    sudo pcimem \
    /sys/devices/pci0000:00/0000:00:03.1/0000:08:00.0/resource2 0x800c w 0x1;
    ```
    Write to enable CMAC1:
    ```
    sudo pcimem \
    /sys/devices/pci0000:00/0000:00:03.1/0000:08:00.0/resource2 0xC014 w 0x1;

    sudo pcimem \
    /sys/devices/pci0000:00/0000:00:03.1/0000:08:00.0/resource2 0xC00c w 0x1;
    ```
### Section 8: Binding DPDK and Testing by Running pktgen

1.  Edit dpdk-devbind.py so that it can find the qdma PCIe class/vendor/device, for example like below.  
        (The file dpdk-devbind.diff in this repo also contains these same lines from diff.)
    ```
    31a32,35
    >
    > qdma = {'Class: '02', 'Vendor': '10ee', 'Device': '903f,913f',
    >               'SVendor': None, 'SDevice': None}
    >
    62c66
    < network_devices = [network_class, cavium_pkx, avp_vnic, ifpga_class]
    ---
    > network_devices = [network_class, cavium_pkx, avp_vnic, ifpga_class, qdma]
    ```

2.  Load dpdk-devbind.py with arguments for vfio and the two pcie bus and device identifiers:  
    ```
    sudo dpdk_patched/dpdk-20.11/usertools/dpdk-devbind.py -b vfio-pci \
	08:00.0 08:00.1
    ```
3.  Test by running pktgen-dpdk (note the command below specifies example device IDs
        and bus IDs for the two PFs, please substitute with the appropriate IDs):
    ```
    sudo dpdk_patched/pktgen-dpdk/usr/local/bin/pktgen -a 08:00.0 -a 08:00.1 \
    -d librte_net_qdma.so -l 4-10 -n 4 -a 00:03.0 -a 00:03.1 -- -m [6:7].0 -m [8:9].1
    ```

---

### Copyright Notice and Disclaimer

© Copyright 2020 – 2022 Xilinx, Inc. All rights reserved.

This file contains confidential and proprietary information of Xilinx, Inc. and is protected under U.S. and international copyright and other intellectual property laws.

DISCLAIMER

This disclaimer is not a license and does not grant any rights to the materials distributed herewith. Except as otherwise provided in a valid license issued to you by Xilinx, and to the maximum extent permitted by applicable law: (1) THESE MATERIALS ARE MADE AVAILABLE "AS IS" AND WITH ALL FAULTS, AND XILINX HEREBY DISCLAIMS ALL WARRANTIES AND CONDITIONS, EXPRESS, IMPLIED, OR STATUTORY, INCLUDING BUT NOT LIMITED TO WARRANTIES OF MERCHANTABILITY, NONINFRINGEMENT, OR FITNESS FOR ANY PARTICULAR PURPOSE; and (2) Xilinx shall not be liable (whether in contract or tort, including negligence, or under any other theory of liability) for any loss or damage of any kind or nature related to, arising under or in connection with these materials, including for any direct, or any indirect, special, incidental, or consequential loss or damage (including loss of data, profits, goodwill, or any type of loss or damage suffered as a result of any action brought by a third party) even if such damage or loss was reasonably foreseeable or Xilinx had been advised of the possibility of the same.

CRITICAL APPLICATIONS

Xilinx products are not designed or intended to be fail-safe, or for use in any application requiring failsafe performance, such as life-support or safety devices or systems, Class III medical devices, nuclear facilities, applications related to the deployment of airbags, or any other applications that could lead to death, personal injury, or severe property or environmental damage (individually and collectively, "Critical Applications"). Customer assumes the sole risk and liability of any use of Xilinx products in Critical Applications, subject only to applicable laws and regulations governing limitations on product liability.

THIS COPYRIGHT NOTICE AND DISCLAIMER MUST BE RETAINED AS PART OF THIS FILE AT ALL TIMES.
