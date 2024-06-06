# Miscellaneous Configuration

This directory contains miscellaneous configuration files required for specific setup, largely related to configuring WiFi APs (access points).

- [`70-ap-interface.rules`](./70-ap-interface.rules):

    Linux creates a wireless interface for each logical radio it detects on boot. However, this station is in 'managed' mode, which is not permitted as a bridge interface child. To workaround this, use a `udev` rule which runs when the interface is created (but before `systemd-networkd` attempts to add it to a bridge). This rule changes the wireless interface type to 'AP' (access point), allowing it to be bridged with other interfaces.

- [`hostapd@.service`](./hostapd@.service)

    While Debian offers this by default, RedHat distributions don't seem to. The default `hostapd.service` file provided on install on both platforms assumes a single interface whose config lies in `/etc/hostapd/hostapd.conf`. This service file provides flexibility, allowing the user to configure multiple `hostapd`-based APs using a single service file. So, on RedHat-based distributions, copy this into `/usr/lib/systemd/system/hostapd@.service`, allowing a user to enable multiple interfaces, for example:

    ```Bash
    sudo systemctl enable hostapd@wlan0.service
    sudo systemctl enable hostapd@wlan1.service
    ```

    The `Before=network.target` and `Wants=network.target` in the `[Unit]` section are also replaced with  `Wants=systemd-networkd-wait-online.network` and `After=systemd-networkd-wait-online.service`. This ensures that `systemd-networkd` renames network interfaces before `hostapd` attempts to use the renamed interfaces.
    
    With the default settings, `hostapd` starts when the network management stack initializes but before `systemd-networkd` renames the interfaces. This causes each `hostapd` service to fail on boot before succeeding on a subsequent attempt (since `Restart=on-failure` is set). The issue is mostly cosmetic but will pollute the logs and potentially cause confusion. See [this `systemd` wiki page](https://systemd.io/NETWORK_ONLINE/) for more details.