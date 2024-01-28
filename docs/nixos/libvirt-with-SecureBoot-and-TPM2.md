# Enabling SecureBoot and TPM2 for libvirt guest VMs on NixOS

The `qemu` package comes with SecureBoot-enabled OVMF. Make this available in a predictable path (we choose `/etc/ovmf`) so it can be referenced from libvirt domain definitions.

Add to your NixOS system configuration (`/etc/nixos/configuration.nix`) to make the firmware files available in a predictable place:

```nix
  environment.etc = {
    "ovmf/edk2-x86_64-secure-code.fd" = {
      source = config.virtualisation.libvirtd.qemu.package + "/share/qemu/edk2-x86_64-secure-code.fd";
    };
    "ovmf/edk2-i386-vars.fd" = {
      source = config.virtualisation.libvirtd.qemu.package + "/share/qemu/edk2-i386-vars.fd";
    };
  };
```

(this assumes that you're running a guest with `x86_64` CPU architecture).

Use the SecureBoot-enabled firmware in the guest by adding the following to the domain configuration XML or updating the existing domain configuration to match:

```xml
<os>
    <type arch='x86_64' machine='pc-q35-6.1'>hvm</type>
    <loader readonly='yes' secure='yes' type='pflash'>/etc/ovmf/edk2-x86_64-secure-code.fd</loader>
    <nvram template='/etc/ovmf/edk2-i386-vars.fd'>/var/lib/libvirt/qemu/nvram/your_domain-OVMF_VARS.fd</nvram>
</os>
```

(replace `your_domain` with the name of your libvirt domain).

You also need to enable System Management Mode in the guest domain XML by enabling the `smm` feature:

```xml
<features>
  <acpi/>
  <apic/>
  <smm state='on'/>
</features>
```

To enable SecureBoot on an existing domain requires you clear your existing VM's NVRAM when you start it:

```shell
virsh start your_domain --reset-nvram
```

You can now manage SecureBoot in your guest. When the VM is booting, hit `ESC` key to enter the OVMF firmware menu and you can configure SecureBoot in the `Device Manager` > `Secure Boot Configuration` menu (e.g. to enter SecureBoot setup mode).

## Adding a TPM2

To make a TPM2 module available to your guest VM, add the following to your domain configuration:

```xml
<tpm>
  <backend type="emulator" version="2.0" />
</tpm>
```

Add the following to your host system's configuration (`/etc/nixos/configuration.nix`):

```nix
virtualisation.libvirtd = {
  qemu = {
    swtpm.enable = true;
  };
};
```

I needed to restart libvirt after adding this otherwise I got an error when trying to update the libvirt domain configuration:

```shell
[root@nixos:/state/vm/kvm]# virsh define your_domain.xml                                                                                                                                            
error: Failed to define domain from your_domain.xml                                                                                                                                                           
error: unsupported configuration: TPM version '2.0' is not supported                                           
```


```shell
[root@nixos:/state/vm/kvm]# systemctl restart libvirtd.service
```

This is sufficient to get a TPM visible in the guest. The TPM can be managed through the OVMF firmware menu (hit `ESC` key when VM is starting up) through the `Device Manager` > `TCG2 Configuration` menu option.

The guest can then be used to set up SecureBoot with optional use of TPM for unattended unlocking of root disk:

```shell
[root@nixos:~]# bootctl
System:
      Firmware: UEFI 2.70 (EDK II 1.00)
 Firmware Arch: x64
   Secure Boot: enabled (user)
  TPM2 Support: yes
  Boot into FW: supported

```


## References

* [libvirt > Secure Boot](https://libvirt.org/kbase/secureboot.html)
* [libvirt > Domain XML format > BIOS bootloader](https://libvirt.org/formatdomain.html#bios-bootloader)