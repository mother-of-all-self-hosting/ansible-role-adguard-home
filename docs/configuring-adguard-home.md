<!--
SPDX-FileCopyrightText: 2020 - 2024 MDAD project contributors
SPDX-FileCopyrightText: 2020 - 2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 - 2025 Suguru Hirahara
SPDX-FileCopyrightText: 2024 MASH project contributors
SPDX-FileCopyrightText: 2024 Sergio Durigan Junior

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up AdGuard Home

This is an [Ansible](https://www.ansible.com/) role which installs [AdGuard Home](https://adguard.com/en/adguard-home/overview.html) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

AdGuard Home is a network-wide DNS software for blocking ads & tracking.

See the project's [documentation](https://adguard.com/kb/) to learn what AdGuard Home does and why it might be useful to you.

> [!WARNING]
> Running a public DNS server is not advisable. You'd better install AdGuard Home in a trusted local network, or adjust its network interfaces and port exposure (via the variables you can find on the section below about [opening ports](#open-ports)) so that you don't expose your DNS server publicly to the whole world. If you're exposing your DNS server publicly, consider restricting who can use it by adjusting the **Allowed clients** setting in the **Access settings** section of **Settings** -> **DNS settings**.

## Prerequisites

### Open ports

You may need to open the following ports on your server:

- `53` over **TCP**, controlled by `adguard_home_container_dns_tcp_bind_port` — used for DNS over TCP
- `53` over **UDP**, controlled by `adguard_home_container_dns_udp_bind_port` — used for DNS over UDP

Docker automatically opens these ports in the server's firewall, so you likely don't need to do anything. If you use another firewall in front of the server, you may need to adjust it.

By default, those ports will be exposed by the container on **all network interfaces**. To expose these ports only on **some** network interfaces, add the following configuration to your `vars.yml` file:

```yaml
# Expose only on 192.168.1.15
adguard_home_container_dns_tcp_bind_port: '192.168.1.15:53'
adguard_home_container_dns_udp_bind_port: '192.168.1.15:53'
```

## Adjusting the playbook configuration

To enable AdGuard Home with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# adguard-home                                                         #
#                                                                      #
########################################################################

adguard_home_enabled: true

########################################################################
#                                                                      #
# /adguard-home                                                        #
#                                                                      #
########################################################################
```

### Set the hostname

To enable AdGuard Home you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
adguard_home_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

>[!WARNING]
> When hosting under a subpath with `adguard_home_path_prefix`, there are quirks caused by [this bug](https://github.com/AdguardTeam/AdGuardHome/issues/5478), such as:
>
> - upon initial usage, you will be redirected to `/install.html` and would need to manually adjust this URL to something like `/adguard-home/install.html` (depending on your `adguard_home_path_prefix`). After the installation wizard completes, you'd be redirected to `/index.html` incorrectly as well.
>
> - every time you hit the homepage and you're not logged in, you will be redirected to `/login.html` and would need to manually adjust this URL to something like `/adguard-home/login.html` (depending on your `adguard_home_path_prefix`)

### Extending the configuration

There are some additional things you may wish to configure about the component.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `adguard_home_environment_variables_additional_variables` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, AdGuard Home becomes available at the specified hostname like `https://example.com`.

To get started, open the URL with a web browser, and follow the set up wizard.

On the set up wizard, **make sure to set the HTTP port under "Admin Web Interface" to `3000`**. This is the in-container port that our Traefik setup expects and uses for serving the install wizard to begin with. If you go with the default (`80`), the web UI will stop working after the installation wizard completes.

Things you should consider doing later:

- increasing the per-client Rate Limit (from the default of `20`) in the **DNS server configuration** section in **Settings** -> **DNS Settings**
- enabling caching in the **DNS cache configuration** section in **Settings** -> **DNS Settings**
- adding additional blocklists by discovering them on [Firebog](https://firebog.net/) or other sources and importing them from **Filters** -> **DNS blocklists**
- reading the AdGuard Home [README](https://github.com/AdguardTeam/AdGuardHome/blob/master/README.md) and [Wiki](https://github.com/AdguardTeam/AdGuardHome/wiki)

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu adguard-home` (or how you/your playbook named the service, e.g. `adguard-home`).

### Workaround for the issue related non-root account

Adguard Home does not currently support being setup with a non-`root` account (see [issue](https://github.com/AdguardTeam/AdGuardHome/issues/4714)). As the playbook uses the user `mash` when starting services, you will likely encounter the following error when `adguard-home.service` tries to start for the first time:

```txt
mar 02 19:11:59 $hostname adguard-home[872496]: 2024/03/02 18:11:59.706251 [info] Checking if AdGuard Home has necessary permissions
mar 02 19:11:59 $hostname adguard-home[872496]: 2024/03/02 18:11:59.706257 [fatal] This is the first launch of AdGuard Home. You must run it as Administrator.
```

You can workaround this issue by editing `adguard-home.service` and temporarily make it start Adguard Home as the `root` user for the first time, and then revert it back to using a regular user afterwards. Follow the steps below, which require you to be `root` to execute the commands:

1. Run `systemctl edit --full adguard-home.service` to edit Adguard Home's service file and remove or comment out the line starting with `--user` (e.g. `--user=996:3992 \` — the numbers represent the uid/gid of the `mash` user, so your values may be different):

    ```txt
    ExecStartPre=/usr/bin/env docker create \
                            --rm \
                            --name=adguard-home \
                            --log-driver=none \
                            --user=996:3992 \  <--- remove temporarily
    ```

2. Run `systemctl restart adguard-home.service` to restart the service.
3. Perform the first time setup as documented under [usage](#usage).
4. Run `systemctl stop adguard-home.service` to stop the service.
5. Run `chown -R mash:mash /mash/adguard-home/workdir` to change ownership of the files created during the first-time setup from `root` to `mash`. Optionally, use `ls -ll /mash/adguard-home/workdir` to check the file ownership before and after running `chown`.
6. Run the playbook again to rebuild `/etc/systemd/system/adguard-home.service` and start AdGuard Home again: `just install-service adguard-home.service`.
7. If you didn't get any errors, Adguard Home should be running correctly. You can also check on the service with: `journalctl -fu adguard-home.service`.
