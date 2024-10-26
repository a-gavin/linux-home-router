# Linux Home Router Configuration

Ansible-based Linux home router configuration, using all open source tooling under the hood (primarily `systemd-networkd`, `nftables`, and `hostapd`).

Once you have verified your configuration matches the tool's [configuration assumptions](#configuration-assumptions) and followed the initial setup outlined in [instructions](#instructions), configuring the system is as simple as running the following (see [this section](#configure-target-machine) for more info):

```Bash
ansible-playbook playbook.yml -i MACHINE, --extra-vars cfg_dir=example_cfgs --ask-become-pass
```

## Configuration Assumptions

In addition to assuming [some Linux networking familiarity](https://a-gavin.github.io/blog/linux-cmds/#querying-network-information), this tool assumes a Fedora target system and a configuration file directory with the following:

- `nftables` configuration file called `nftables.conf`
- `systemd-networkd` configuration files with suffix `.network` in sub-directory called `systemd-networkd/`
  - Assume one `.network` file (likely a LAN bridge) configures the LAN DHCP server
  - If exist, configuration files with suffix `.link` and `.netdev` will be installed as well
- OPTIONAL: `hostapd` configuration file(s) in sub-directory called `hostapd/`
  - `hostapd` configuration files should be of the format `interface.conf`, where `interface` is the name of the wireless interface to be used as an access point (e.g. `wlan0`)

## Instructions

**WARNING:** It is strongly suggested to have non-networked access to the target router system (e.g. monitor and keyboard or serial). It's possible (even likely) you will lose network access to the machine in the process.

Ansible install steps should be performed on the machine which you intend to run Ansible on (not necessarily the target router machine, although that's possible too).

In my setup, I have have Ansible installed on my workstation where I store my configuration and this repository. The target router machine has its wired WAN port on the same network, so I can access it (connecting to the system using any active WiFi AP interfaces will fail during the Ansible playbook run). I use a separate, third machine to validate the target router machine's LAN to WAN connectivity. Depending on your `nftables` rules, you may have to stop `nftables` on the target router machine when re-provisioning the system (Ansible uses SSH and you will hopefully disable SSH on your WAN interface).

### Install Ansible

1. Install `pipx`

   NOTE: `pipx` is a helper tool which installs Python-based programs
   in an isolated virtual environment and adds them to your `$PATH`.
   This allows you to run installed Python programs by their name
   instead of `python -m XXX` while also isolating the program's
   dependencies from the system's Python installation.

   ```Bash
   # Fedora
   sudo dnf install -y pipx

   # Ubuntu
   sudo apt install -y pipx
   ```

2. Add `$HOME/.local/bin` to your `PATH` environment variable, if it isn't already.

   The best place to do this is in your shell's config file (e.g. `~/.bashrc`, `~/.zshrc`). You can either do this manually or by running `pipx ensurepath`.

   When you install Python programs with `pipx`, the program executables will be available in `$HOME/.local/bin`. Because `pipx` installs to a directory in your `$HOME` directory tree, other users will not have Ansible installed unless they perform the same steps. Packages installed with package managers like `apt` and `dnf` instead install to system-level directories like `/usr/bin` which should be on your `PATH` by default.

3. Install `ansible`

   ```Bash
   pipx install --include-deps ansible
   ```

4. Install `ansible` dependencies/plugins

   ```Bash
   pipx inject ansible argcomplete

   # Required for disabling SELinux and configuring services
   ansible-galaxy collection install ansible.posix
   ```

### Configure `nftables` and `systemd-networkd`

**NOTE:** Configuring these tools was the most time consuming portion for me. Aside from simple things like ensuring things match, you may also run into headaches like `systemd-networkd` case sensitivity and confusion with the different types of `systemd-networkd` configuration files and sections.

Simple configuration is provided in [`example_cfgs/`](./example_cfgs/). It describes a system with a DHCP-receiving WAN interface and a bridged LAN interface serving DHCP for a `192.168.150.0/24` subnet. This configuration permits only rate-limited ICMP on the WAN interface and only SSH, DHCP, and DNS on the LAN interface. NAT forwarding is allowed only from the source addresses on the subnet LAN.

The WAN interface is hardcoded to `wan0`. The bridged LAN interface bridges a LAN Ethernet interface `lan0` and LAN WiFi interface `wlan0`. Your interface names will likely be different, but you can rename them using a `systemd-networkd` configuration file like [`10-lan0.link`](./example_cfgs/systemd-networkd/10-lan0.link). See the [`systemd.network` documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.network.html) for more information.

I found the following documentation to be useful:

- `nftables`:
  - [`nftables` Home Router Example](https://wiki.nftables.org/wiki-nftables/index.php/Simple_ruleset_for_a_home_router)
      - The example `nftables` ruleset is largely based on this
  - [`nftables` Quick Reference](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)
  - [`nftables` Ruleset Debugging/Tracing](https://wiki.nftables.org/wiki-nftables/index.php/Ruleset_debug/tracing)
- `systemd-networkd`
  - [`man systemd.link`](https://www.freedesktop.org/software/systemd/man/latest/systemd.link.html)
  - [`man systemd.netdev`](https://www.freedesktop.org/software/systemd/man/latest/systemd.netdev.html)
  - [`man systemd.network`](https://www.freedesktop.org/software/systemd/man/latest/systemd.network.html)
  - [SuperUser: How to debug `systemd-networkd`](https://superuser.com/questions/1187633/how-to-debug-systemd-networkd)

### Configure Target Machine

Use the `ansible-playbook` command to configure the machine `MACHINE`, installing configuration available in the `example_cfgs/` directory.

```Bash
# Config directory is './example_cfgs/', but you can change this
# to whatever you want so long as you update the 'cfg_dir=' portion
# of the command
ansible-playbook playbook.yml -i MACHINE, --extra-vars cfg_dir=example_cfgs --ask-become-pass
```

Notes:

- Due to limitations in `ansible-playbook`, you must specify the machine with a comma immediately following the IP address or hostname. Otherwise, `ansible-playbook` will interpret the value as a file to read. See the [official documentation](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html#cmdoption-ansible-playbook-i) for more information.
- Option `--ask-become-pass` will prompt you for the password for the SSH user on the target system to perform tasks which require root permissions (e.g. configuring services).

## Other Useful Commands

- Upgrade `ansible`

  ```Bash
  pipx upgrade --include-injected ansible
  ```

- List `pipx` packages (and injected dependencies)
  ```Bash
  pipx list
  pipx list --include-injected
  ```

## Development Setup

The following steps detail how to install and use Vagrant for basic configuration testing. For testing `nftables` rules and `systemd-networkd` interface configuration, a physical machine is required.

1. Follow [instructions to install `ansible` and required plugins above](#install-ansible)

2. Install `libvirt`, `vagrant`, and related tools

   Program `virt-manager` is a GUI frontend for configuring `libvirt` VMs. It is primarily useful here to view and access the system. These instructions assume you will use `vagrant` to configure the `libvirt` VM.

   NOTE: NFS server package should be installed when `libvirt` is installed on both Fedora and Ubuntu, but add it to the list of packages for posterity's sake.

   ```Bash
   # Fedora host

   # Ubuntu host
   sudo apt install -y \
       qemu-kvm libvirt-daemon-system virt-manager vagrant-libvirt
   ```

3. Add your user to the `libvirt` group

   NOTE: Either re-login or run `newgrp libvirt` for changes to take effect. Running `newgrp` will only update the current shell.

   ```Bash
   sudo usermod -a -G libvirt $USER
   ```

4. Install Vagrant plugins

   ```Bash
   vagrant plugin install vagrant-env
   ```

5. (OPTIONAL) Install, start, and enable NFS (faster VM synced directories than sshfs)

   ```Bash
   # Fedora
   # See: https://developer.fedoraproject.org/tools/vagrant/vagrant-nfs.html
   sudo dnf install -y nfs-utils
   sudo systemctl enable nfs-server && sudo systemctl start nfs-server

   # Ubuntu
   sudo apt install -y nfs-kernel-server
   sudo systemctl enable nfs-server && sudo systemctl start nfs-server
   ```

   - Ubuntu requires you to explicitly enable `NFS` UDP in `/etc/nfs.conf`. You can do this by adding modifying the `[nfsd]` section to contain `udp=y` on its own line and then running `sudo systemctl restart nfs-server`.

   - If running `firewalld` on Fedora, you will also need to run the following commands:

     ```Bash
     sudo firewall-cmd --permanent --zone=libvirt --add-service=nfs3 \
        && sudo firewall-cmd --permanent --zone=libvirt --add-service=nfs \
        && sudo firewall-cmd --permanent --zone=libvirt --add-service=rpc-bind \
        && sudo firewall-cmd --permanent --zone=libvirt --add-service=mountd \
        && sudo firewall-cmd --reload
     ```

6. Start the VM, specifying the target operating system and config directory with the `VM_OS` and `CFG_DIR` environment variables, respectively. Only Ubuntu 22 (`u22`) and Fedora 40 (`f40`, default) are supported at this time.

   For example:

   ```Bash
   CFG_DIR=example_cfgs VM_OS=f40 vagrant up
   ```
