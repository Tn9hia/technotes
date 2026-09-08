# What is cloud init?
>Cloud-init là một công cụ quan trọng được sử dụng trong các hệ điều hành Linux cloud để tự động cấu hình và quản lý các máy ảo (VM) ngay từ khi chúng khởi động lần đầu. 

- [user-data](https://cloudinit.readthedocs.io/en/latest/explanation/format.html#user-data-formats) is provided by the user, and cloud-init recognizes many different formats.
- [vendor-data](https://cloudinit.readthedocs.io/en/latest/explanation/vendordata.html#vendor-data) is provided by the cloud provider.
- [meta-data](https://cloudinit.readthedocs.io/en/latest/explanation/instancedata.html#instance-data) contains the platform data, including things like machine ID, hostname, etc.

Cloud-init sử dụng tệp cấu hình `user-data` để chỉ định các hành động khi khởi động. Một tệp mẫu thường dùng có dạng YAML như sau:
```yaml
#cloud-config
hostname: example-host
users:
  - name: admin
    ssh-authorized-keys:
      - ssh-rsa AAAAB3... user@domain
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
write_files:
  - path: /etc/motd
    content: |
      Welcome to the server!
runcmd:
  - apt-get update
  - apt-get install -y nginx

```


# How to guide
## Validate user-data
```sh
sudo cloud-init schema --system --annotate
# or
cloud-init schema -c test.yml --annotate
```

## Debug cloud init
### Cloud init does not run 
1. Check the status of `cloud-init status --long`
2. Check log file `/run/cloud-init/ds-identify.log`
3. Check the status of service
```shell
systemctl status cloud-init-local.service \
cloud-init-network.service \
cloud-config.service \
cloud-final.service
```
### Cloud init run but the result does not as my expectation
## Cloud init status

```
"not started"
"running"
"done"
"error - done"
"error - running"
"degraded done"
"degraded running"
"disabled"
```
### Cloud-init enablement status

- `'unknown'`: `ds-identify` has not run yet to determine if cloud-init should be run during this boot
- `'disabled-by-marker-file'`: `/etc/cloud/cloud-init.disabled` exists which prevents cloud-init from ever running
- `'disabled-by-generator'`: `ds-identify` determined no applicable cloud-init datasources
- `'disabled-by-kernel-command-line'`: kernel command line contained cloud-init=disabled
- `'disabled-by-environment-variable'`: environment variable `KERNEL_CMDLINE` contained `cloud-init=disabled`
- `'enabled-by-kernel-command-line'`: kernel command line contained cloud-init=enabled
- `'enabled-by-generator'`: `ds-identify` detected possible cloud-init datasources
- `'enabled-by-sysvinit'`: enabled by default in SysV init environment

## Re-run cloud-init
Most cloud-init configuration is only applied to the system once. This means that simply rebooting the system will only re-run a subset of cloud-init. Cloud-init provides two different options for re-running cloud-init for debugging purposes.
### Remove the logs and cache, then reboot
```shell
cloud-init clean --logs --reboot
```

### Run a single cloud-init module
```shell
sudo cloud-init single --name cc_ssh --frequency always
```
### Manually run cloud-init stages
```shell
cloud-init --all-stages
```

## Disable cloud-init
**Method 1**: Create a empty text file
```shell
touch /etc/cloud/cloud-init.disabled
```

**Method 2**:  kernel command line

```shell
echo 'GRUB_CMDLINE_LINUX="cloud-init=disabled"' >> /etc/default/grub
grub-mkconfig -o /boot/efi/EFI/ubuntu/grub.cfg
```

**Method 3**: environment variable
```shell
echo "DefaultEnvironment=KERNEL_CMDLINE=cloud-init=disabled" >> /etc/systemd/system.conf
```


## Cloud-init cli
**Analyze**: Get detailed reports of where `cloud-init` spends its time during the boot process.
**Clean**: Remove `cloud-init` artifacts from `/var/lib/cloud` and config files (best effort) to simulate a clean instance.
- **--log**: - Optionally remove all `cloud-init` log files in `/var/log/`.
- **--reboot**: Reboot the system after removing artifacts.
- **--machine-id**: Set `/etc/machine-id` to `uninitialized\n` on this image for systemd environments.
- **--configs [all | ssh_config | network | datasource | fstab ]**: Optionally remove all `cloud-init` generated config files.
- - **--seed**: Remove the cloud-init seed directory (e.g., `/var/lib/cloud/seed/`) which stores instance metadata used initializing a datasource. Useful when regenerating metadata from a new or updated seed source.
 **status**: Report cloud-init’s current status.
References
- [cloud-init 24.3.1 documentation (cloudinit.readthedocs.io)](https://cloudinit.readthedocs.io/en/latest/)
