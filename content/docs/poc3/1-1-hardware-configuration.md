---
title: "1.1 Hardware Requirement and Configuration"
---

Phala Network operates on trusted hardwares, in this case, Intel CPUs which support SGX feature. SGX was first introduced in 2015 with the sixth generation Intel Core microprocessors based on the Skylake microarchitecture. In practice, the support of SGX feature also relies on your motherboard and BIOS configurations.
In this section, we will first ensure that necessary hardware requirements are satisfied and all your hardwares are correcly configured.

## General Hardware Requirements
![](/images/docs/poc3/1.2.png)

### CPU Requirement

You can refer to the following tutorial to ensure that your CPU is SGX-supported.

- [Whether your CPU is SGX-supported](https://forum.phala.network/t/how-to-check-whether-your-cpu-is-sgx-supported/1252)

### BIOS Settings

First of all, we recommand to update your BIOS to the latest version.

1. **Enter BIOS**. The method varies for different motherboards, and Google is always at your service about this.
3. **Disable Secure Boot**. Go to `Security` -> `Secure Boot`, set it to `Disabled`.
4. **Enable SGX Extension**. Go to `Security` -> `SGX` (This name  may vary according to the different manufacturers), set it to `Enabled`.
>If you can only find `SGX: Software Controlled` option, you will have to run [sgx-software-enable](https://github.com/intel/sgx-software-enable) in Ubuntu. You can follow the Intel's instructions, build it from source and execute it. Also, we provide a prebuilt file for Ubuntu 18.04 / 20.04 that can be found [here](https://github.com/Phala-Network/sgx-tools/releases/tag/0.1). You can download and execute it with the following commands:
> ```bash
> wget https://github.com/Phala-Network/sgx-tools/releases/download/0.1/sgx_enable
> chmod +x sgx_enable 
> sudo ./sgx_enable
> ```
5. **Use UEFI Boot**. Go to `Boot` -> `Boot Mode`, and make sure it was set to `UEFI`.
6. Save and reboot.

You also need to make sure that Ubuntu is installed in UEFI mode. SGX is not guaranteed to work properly on the OS installed in legacy mode. In such case you may want to reinstall the system.

## SGX Driver Installation

Intel provides two kinds of SGX drivers: SGX driver and DCAP driver. The latter one is preferred since it provides more security-related features which can be utilized by Phala in the future, while the former one provides better compatibility and supports old CPUs without FLC support. For now, both drivers work for Phala.

First, check if you already have either of the drivers installed:

- Type in command `ls /dev/isgx`. If the file exists, you have the SGX driver installed
- Type in command `ls /dev/sgx`. If the directory exists, you have the DCAP driver installed

As long as either type of driver works, your computer will be ready to mine PHA. If neither of the types works, please try to install the DCAP driver first. Type in the commands below to install DCAP driver:

```bash
sudo apt-get install dkms
wget https://download.01.org/intel-sgx/sgx-dcap/1.9/linux/distro/ubuntu18.04-server/sgx_linux_x64_driver_1.36.2.bin
chmod +x sgx_linux_x64_driver_1.36.2.bin
sudo ./sgx_linux_x64_driver_1.36.2.bin
```

After installation, retry `ls /dev/sgx` to check whether the installation succeeds. If the DCAP driver doesn't work, please **FIRST UNINSTALL DCAP DRIVER** and then switch to the SGX driver. To uninstall the DCAP driver, run:

```bash
sudo /opt/intel/sgxdriver/uninstall.sh
```

Then try following commands to install the SGX driver:

```bash
wget https://download.01.org/intel-sgx/sgx-linux/2.12/distro/ubuntu18.04-server/sgx_linux_x64_driver_2.11.0_4505f07.bin
chmod +x sgx_linux_x64_driver_2.11.0_4505f07.bin
sudo ./sgx_linux_x64_driver_2.11.0_4505f07.bin
```

Run `ls /dev/isgx` again to check if the SGX driver works.

## Double check the SGX capability

After the installation of your driver, please use the following utility to double check if everything goes well. Docker is required for this step, and you can peek at how to install it in the [next secton]({{< relref "docs/poc3/1-2-software-configuration" >}}).

- If you are using the DCAP driver:

  ```bash
  docker run -ti --rm --name phala-sgx_detect --device /dev/sgx/enclave --device /dev/sgx/provision phalanetwork/phala-sgx_detect
  ```

- If you are using the SGX driver:

  ```bash
  docker run -ti --rm --name phala-sgx_detect --device /dev/isgx phalanetwork/phala-sgx_detect
  ```

- Even if there's no `/dev/sgx` or `/dev/isgx`, you can still run the utility to collect some diagnose information:

  ```bash
  docker run -ti --rm --name phala-sgx_detect phalanetwork/phala-sgx_detect
  ```

Please pay attention to the following checks:

1. SGX system software → Able to launch enclaves → `Production Mode`
2. Flexible launch control → `Able to launch production mode enclave`

Among them, **the former one is a must to run Phala Network pRuntime**. If it's not supported (tagged as ✘ in the report example below), we are afraid you can't mine PHA with this setup. You may want to replace the motherboard and/or the CPU.

The latter is not a must though, it is suggested to be checked as it would be essential to install the DCAP driver.

The report below would be a positive result:

```txt
Detecting SGX, this may take a minute...
aesm_service[15]: Malformed request received (May be forged for attack)
✔  SGX instruction set
  ✔  CPU support
  ✔  CPU configuration
  ✔  Enclave attributes
  ✔  Enclave Page Cache
  SGX features
    ✘  SGX2  ✘  EXINFO  ✘  ENCLV  ✘  OVERSUB  ✘  KSS
    Total EPC size: 94.0MiB
✘  Flexible launch control
  ✔  CPU support
  ？ CPU configuration
  ✘  Able to launch production mode enclave
✔  SGX system software
  ✔  SGX kernel device (/dev/isgx)
  ✔  libsgx_enclave_common
  ✔  AESM service
  ✔  Able to launch enclaves
    ✔  Debug mode
    ✘  Production mode
    ✔  Production mode (Intel whitelisted)

🕮  Flexible launch control > CPU configuration
Your hardware supports Flexible Launch Control, but whether it's enabled could not be determined. More information might be available by re-running this program with sudo. Would you like to do that?
(not supported yet)

debug: Error reading MSR 3Ah: Failed to load MSR kernel module
debug: cause: Failed to load MSR kernel module
debug: cause: Failed executing modprobe
debug: cause: No such file or directory (os error 2)

More information: https://edp.fortanix.com/docs/installation/help/#flc-cpu-configuration

🕮  SGX system software > Able to launch enclaves > Production mode
The enclave could not be launched. This might indicate a problem with FLC.

debug: failed to load report enclave
debug: cause: failed to load report enclave
debug: cause: The EINITTOKEN provider didn't provide a token
debug: cause: aesm error code GetLicensetokenError_6

More information: https://edp.fortanix.com/docs/installation/help/#run-enclave-prod

You're all set to start running SGX programs!
Generated machine id:
[162, 154, 220, 15, 163, 137, 184, 233, 251, 203, 145, 36, 214, 55, 32, 54]

Testing RA...
aesm_service[15]: [ADMIN]EPID Provisioning initiated
aesm_service[15]: The Request ID is 09a2bed647d24f909d4a3990f8e28b4a
aesm_service[15]: The Request ID is 8d1aa4104b304e12b7312fce06881260
aesm_service[15]: [ADMIN]EPID Provisioning successful
isvEnclaveQuoteStatus = GROUP_OUT_OF_DATE
platform_info_blob { sgx_epid_group_flags: 4, sgx_tcb_evaluation_flags: 2304, pse_evaluation_flags: 0, latest_equivalent_tcb_psvn: [15, 15, 2, 4, 1, 128, 6, 0, 0, 0, 0, 0, 0, 0, 0, 0, 11, 0], latest_pse_isvsvn: [0, 11], latest_psda_svn: [0, 0, 0, 2], xeid: 0, gid: 2919956480, signature: sgx_ec256_signature_t { gx: [99, 239, 225, 171, 96, 219, 216, 210, 246, 211, 20, 101, 254, 193, 246, 66, 170, 40, 255, 197, 80, 203, 17, 34, 164, 2, 127, 95, 41, 79, 233, 58], gy: [141, 126, 227, 92, 128, 3, 10, 32, 239, 92, 240, 58, 94, 167, 203, 150, 166, 168, 180, 191, 126, 196, 107, 132, 19, 84, 217, 14, 124, 14, 245, 179] } }
advisoryURL = https://security-center.intel.com
advisoryIDs = "INTEL-SA-00219", "INTEL-SA-00289", "INTEL-SA-00320", "INTEL-SA-00329"
```

If your got a report like below, please screenshot it, and send it to [Phala Discord Server](https://discord.gg/zjdJ7d844d) or [Telegram Miner Group](https://t.me/phalaminer) for help.

```txt
Detecting SGX, this may take a minute...
✔  SGX instruction set
  ✔  CPU support // if tagged with ❌: it does not suppoort SGX function, you would need to use other types of CPU.
  ✔  CPU configuration // if tagged with ❌: you would need to check BIOS updates.
  ✔  Enclave attributes // if tagged with ❌: probably caused by [CPU support issue] and [CPU configuration]
  ✔  Enclave Page Cache // if tagged with ❌: probably caused by [CPU support issue] and [CPU configuration]
  SGX features
    ✘  SGX2  ✘  EXINFO  ✘  ENCLV  ✘  OVERSUB  ✘  KSS // It's OK if SGX2 was tagged with ❌. Phala has not integrated with SGX2 technology in the current stage.
    Total EPC size: 94.0MiB
✘  Flexible launch control
  ✔  CPU support
  ✘  CPU configuration // if tagged with ❌: you can give it a try but your miner might be affected when the SGX driver upgrades in the future.
✔  SGX system software
  ✔  SGX kernel device (/dev/isgx)
  ✔  libsgx_enclave_common
  ✔  AESM service
  ✔  Able to launch enclaves
    ✔  Debug mode
    ✘  Production mode // if tagged with ❌: you would need to check BIOS updates.
    ✔  Production mode (Intel whitelisted)

🕮  Flexible launch control > CPU configuration
Your hardware supports Flexible Launch Control, but it's not enabled in the BIOS. Reboot your machine and try to enable FLC in your BIOS. Alternatively, try updating your BIOS to the latest version or contact your BIOS vendor.

debug: MSR 3Ah IA32_FEATURE_CONTROL.SGX_LC = 0

More information: https://edp.fortanix.com/docs/installation/help/#flc-cpu-configuration

🕮  SGX system software > Able to launch enclaves > Production mode
The enclave could not be launched. This might indicate a problem with FLC.

debug: failed to load report enclave
debug: cause: failed to load report enclave
debug: cause: The EINITTOKEN provider didn't provide a token
debug: cause: aesm error code GetLicensetokenError_6
```

If you can't run Phala pRuntime with both of them tagged as ✔, you may have to check whether your BIOS is the latest version with latest security patches. If you still can't run Phala pRuntime docker with the latest BIOS of your motherboard manufacturer, we are afraid you can't mine PHA for now with this motherboard.

---

#### Reference

1. [What is Trusted Execution Environment](https://www.trustonic.com/technical-articles/what-is-a-trusted-execution-environment-tee/)
2. [What is SGX](https://software.intel.com/content/www/us/en/develop/topics/software-guard-extensions.html)

#### Miner Community

[![](https://img.shields.io/discord/697726436211163147?label=Phala%20Discord)](https://discord.gg/zzhfUjU) [![](https://img.shields.io/badge/Join-Telegram-blue)](https://t.me/phalaminer)
