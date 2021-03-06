# Installing Ubuntu in dual-boot configuration on New Razer Blade for deep learning.

This guide shows how to install Ubuntu 16.10 on New Razer Blade with 1060 NVIDIA GPU to use for deep learning. That means the NVIDIA GPU **cannot** be used for any graphics workloads, it will run only CUDA applications, including deep learning toolkits like Caffe, TensorFlow etc.

**Note**: such configuration also brings some inconveniences, like inability to use HDMI port in Ubuntu: in new Razer, HDMI port is connected directly to 1060 GPU so if the GPU is disabled for graphics, you won't be able to use it for external display either. To connect external display, use USB-C port on Razer (with appropriate USB-C -> HDMI adapter if needed).

The guide assumes UEFI and Secure Boot are **enabled**. Ubuntu works just fine with both of them, no need to disable any of these features.

1. Download Ubuntu 16.10 ISO image and create bootable USB by following instructions:
https://www.ubuntu.com/download/desktop/create-a-usb-stick-on-windows
Note that you must download at least 16.04.**2** or later otherwise (e.g. with just 16.04) many things will not work (WiFi, display resolution etc).
The steps below assume verion 16.10 (note that 16.10 is **not** LTS release!).

2. Boot from USB.

3. Prepare the disk for Ubuntu and install.
    1. Run ```gparted``` and resize the Windows partition as needed. Ideally this should be done as soon as possible, before installing any stuff on Windows.
    2. Install Ubuntu from USB stick. Reboot after install.

4. Post-install steps.
    1. To avoid time issues (GTC/local) due to dual boot, do this:
        Check what time is currently used by Ubuntu:
        ```
        timedatectl | grep local
        ```
        Set it to use local time instead of UTC:
        ```
        timedatectl set-local-rtc 1
        ```
        And check again.
        See http://askubuntu.com/a/720466 for more details.

    2. When switching to terminal (Ctrl+Alt+F1 etc) you may see annoying messages like ```PCIe Bus Error: severity=Corrected```.
        In this case, update your GRUB config (```/etc/default/grub```) according to this: https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1521173
        While you at editing GRUB, change GRUB resolution to native by uncommenting and changing this line:
        ```
        GRUB_GFXMODE=1920x1080
        ```
        And change GRUB background to a nice color:
        ```
        sudo nano /usr/share/plymouth/themes/ubuntu-logo/ubuntu-logo.grub
        ```
        (use ```0,0,0,0``` to set black color)
    
    Don't forget to run ```sudo update-grub``` after making the changes.

5. Optional: install ```inxi``` utility:
    ```
    sudo apt install inxi
    ```
    Running ```inxi -G``` will output current graphics configuration. Note that Intel GPU is not properly recognized, e.g.:
    ```
    Graphics:  Card-1: Intel Device 591b
               Card-2: NVIDIA GP106M [GeForce GTX 1060]
               Display Server: X.Org 1.18.4 drivers: (unloaded: fbdev,vesa)
               Resolution: 1920x1080@60.05hz
               GLX Renderer: Mesa DRI Intel Kabylake GT2 GLX Version: 3.0 Mesa 12.0.6
    ```
    Run ```inxi -F``` to see more information about the system.

6. To improve performance and power characteristics of Intel CPU/GPU on Ubunut, install Intel drivers from: 
    https://01.org/linuxgraphics/downloads/update-tool

    Make sure to download the right version (2.0.2 for 16.04, 2.0.4 for 16.10) otherwise the installation might not succeed.
    ```
    sudo dpkg -i intel-graphics-update-tool_2.0.4_amd64.deb
    ```
    If it fails with missing dependencies, run
    ```
    sudo apt-get -f install
    ```
    and then repeat installation.

7. The most complicated part: install NVIDIA drivers. Make sure you read and understand it first! Download and install the latest version of the drivers and then:
    1. Switch to terminal (Ctrl+Alt+F1):
        
        ```
        sudo service lightdm stop
        sudo NVIDIA_XXX.run --no-opengl-files
        ```

        **Note**: ```--no-opengl-files``` option essentially disables using NVIDIA GPU with X manager (graphics). CUDA applications and deep learning toolkits that use GPU will still work just fine.
        
        During installation, make sure to create public key to sign the driver otherwise it won't load due to secure boot/UEFI.
        Note the location and the name of the key (.DER/.KEY files). Also, select "No" when asked whether to delete private key (you will need it in case you will update drivers in the future). Since the private key remains on the disk, you might want to take some safety precautions (google it for more details).
        
        Reboot. Try running ```nvidia-smi```, it should fail with something like ```module not loaded``` error. This is expected as we haven't enrolled certificates to secure boot yet.
    2. Now import the key using mokutil tool:
        ```
        sudo mokutil --import /usr/share/nvidia/nvidia-modsign-crt-XXX.der
        ```
        Provide a passsword and make sure to remember it!
    
        Reboot. In the MOK screen, enroll the key, provide the password from the previous step when requested.
        Now run ```nvidia-smi``` again, it should work and display proper information about the GPU.
        
        Note: if for some reason you need to delete the key(s) from secure boot, do the following:
        First, list all currently enrolled certificates:
        ```
        mokutil --list-enrolled
        ```
        Remeber the index of the certificate you want to delete.
        Next, export certificates:
        ```
        mokutil --export
        ```
        This will create files with names like MOK-0001.der etc.
        Delete the certificate:
        ```
        sudo mokutil --delete MOK-000X.der
        ```
        (replace X with the index of the cert you want to delete). Now reboot and finish removal process in MOK screen.

    **Important**: to update the driver, use the same key pair to avoid re-generating and enrolling every time.
    To use already existing keys, run NVIDIA driver package with path to public and private keys, something like that:
    ```
    sudo ./NVIDIA-Linux-x86_64-378.13.run --no-opengl-files --module-signing-secret-key=/usr/share/nvidia/nvidia-modsign-key-XXX.key --module-signing-public-key=/usr/share/nvidia/nvidia-modsign-crt-XXX.der
    ```
    Note: you can use the same keys to sign other kernel modules, see Appendix.
8. Now reboot and check the Windows boot is still working (it should :) ).

### That's it!

#### Appendix

##### Installing kernel modules in secure boot
Installing kernel modules (e.g. drivers) can be a bit tricky when secure boot is enabled. Next section demonstrates how to install a driver for USB WiFi AC adapter (AlfaNetworks AWUS036AC). Installing with ```sudo apt install rtl8812au-dkms``` currently [fails](https://bugs.launchpad.net/ubuntu/+source/rtl8812au/+bug/1629235) on 16.04 with kernel 4.8 and later, so alternative method of building a kernel module from sources and installing using DKMS is required. 
1. Get latest version of the driver from the [repo](https://github.com/abperiasamy/rtl8812AU_8821AU_linux).
2. Build and install with DKMS:
    ```
    sudo make -f Makefile.dkms install
    ```
    Run ```sudo modprobe rtl8812au``` - you should see an error message that module is not loaded due to missing key.
3. Sign the module using the key created earlier for NVIDIA driver:
    ```
    sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 /path_to_KEY_file/nvidia-modsign-key-XXX.key /path_to_DER_file/nvidia-modsign-crt-XXX.der /lib/modules/$(uname -r)/updates/dkms/rtl8812au.ko
    ```
4. Reboot and run ```sudo modprobe rtl8812au``` again - it should work fine now (adapter should work fine as well).

Note that it might be necessary to re-install the driver after the kernel updates in case the kernel version changes otherwise the device will not work properly.
The previous version of the driver needs to be removed from DKMS:
1. Check the driver version currently registered in DKMS:
```
sudo dkms status
```
You might see something like:
```
rtl8812au, 4.3.14, 4.8.0-38-generic, x86_64: installed
rtl8812au, 4.3.14, 4.8.0-39-generic, x86_64: installed
```
2. Uninstall the current version of the driver:
```
sudo dkms remove rtl8812au/4.3.14 --all
```
Check that driver was uninstalled using ```dkms status``` again.
3. Rebuild and re-sign the driver again by following previously described steps.

Annoying, yes...

