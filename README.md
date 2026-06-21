# proxmox_ct - Proxmox Container Role

This Ansible role creates and configures a Proxmox Container (LXC).

## Description

This role creates a new Container on a Proxmox server using the `community.proxmox` collection.

## Requirements

- Proxmox VE server with API access
- Ansible collection `community.proxmox` installed
- SSH key pair for container access

## Dependencies

```shell
ansible-galaxy collection install community.proxmox
```

## Role Variables

### Proxmox API Connection Variables

These should be defined in your inventory or playbook to connect to the Proxmox server:

- `proxmox_api_host`: Proxmox API URL or IP address
- `proxmox_api_user`: Proxmox API user (e.g., `root@pam!ansible`)
- `proxmox_api_token_id`: Proxmox API token ID (if using token)
- `proxmox_api_token_secret`: Proxmox API token secret (if using token)
- `proxmox_api_password`: Proxmox API password (if not using token)
- `proxmox_node`: Proxmox node name (e.g., `pve`)

### Container Configuration Variables

The following variables map directly to the `community.proxmox.proxmox` module parameters or control the role's execution.

#### General Configuration

- `pct_vmid`: Container VM ID (e.g., `104`). If not specified, the next available ID is used or it is discovered based on hostname.
- `pct_hostname`: Container hostname (e.g., `container-01`).
- `pct_description`: Container description shown in the Proxmox UI.
- `pct_pool`: Add the new container to the specified Proxmox resource pool.
- `pct_tags`: List of tags to apply to the container (e.g., `['web', 'prod']`).
- `pct_state`: Desired state of the container (`present`, `started`, `stopped`, `absent`, `restarted`, `template`). Default is `present`.
- `pct_force`: Force operations (e.g., overwrite existing container if `state=present`).
- `pct_update`: If true, the container will be updated with new values (only for `state=present`).
- `pct_purge`: Remove container from all related configurations (backup, replication, HA) when `state=absent`.
- `pct_delete`: A list of settings to delete from the container configuration.
- `pct_timeout`: Timeout for operations in seconds.

#### OS & Template Configuration

- `pct_ostemplate`: The OS template to use for creation (e.g., `local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst`).
- `pct_ostype`: OS type (e.g., `debian`, `ubuntu`, `centos`, `alpine`).
- `pct_clone`: ID of the container to be cloned (instead of using an OS template).
- `pct_clone_type`: Type of the clone created (`full`, `linked`, `opportunistic`).

#### Resources (CPU & Memory)

- `pct_cores`: Number of CPU cores per socket.
- `pct_cpus`: Number of allocated CPUs.
- `pct_cpuunits`: CPU weight for the container.
- `pct_memory`: RAM size in MB.
- `pct_swap`: Swap memory size in MB.

#### Storage & Disks

- `pct_storage`: Target storage backend for the rootfs (e.g., `local-lvm`). Mutually exclusive with `pct_disk_volume`.
- `pct_disk`: Legacy rootfs disk configuration string.
- `pct_disk_volume`: Dictionary configuring the rootfs disk (e.g., `storage`, `size`, `host_path`).
- `pct_mount_volumes`: List of dictionaries defining additional mount points (e.g., `id`, `mountpoint`, `storage`, `size`).
- `pct_mounts`: Dictionary defining mount points as strings.

#### Network & DNS

- `pct_netif`: Dictionary configuring network interfaces using raw Proxmox configuration strings (e.g., `net0: "name=eth0,ip=dhcp,bridge=vmbr0"`).
- `pct_netif_dict`: A structured alternative to `pct_netif`. A dictionary of dictionaries where network interface options are specified as key-value pairs (e.g., `bridge`, `ip`, `gw`, etc.). It dynamically renders into the format required by Proxmox.
- `pct_ip_address`: Specifies the address the container will be assigned (if not using `pct_netif` or `pct_netif_dict`).
- `pct_nameserver`: DNS server IP address for the container.
- `pct_searchdomain`: DNS search domain for the container.

#### Security & Access

- `pct_password`: Root password for the container.
- `pct_pubkey`: Public SSH key to add to `/root/.ssh/authorized_keys`.
- `pct_unprivileged`: Run as an unprivileged container (boolean). Default is true in newer module versions.
- `pct_features`: List of features to enable (e.g., `['nesting=1', 'keyctl=1']`).
- `pct_timezone`: Timezone used by the container (e.g., `Europe/Paris`, or special value `host`).

#### Boot & Lifecycle

- `pct_onboot`: Start the container automatically during system bootup (boolean).
- `pct_startup`: Specifies the startup/shutdown order and delays.
- `pct_hookscript`: Script that will be executed during various steps in the container's lifetime.

#### Custom Configuration

- `pct_extra_config`: A list of dictionaries (using `lineinfile` parameters) to inject raw custom lines into `/etc/pve/lxc/{{ pct_vmid }}.conf` directly.

## Example Playbook

```yaml
---
- hosts: proxmox_hosts
  vars:
    proxmox_api_host: "pve.example.com"
    proxmox_api_user: "root@pam!ansible"
    proxmox_api_token_id: "ansible-token"
    proxmox_api_token_secret: "your-api-token-secret"
    proxmox_node: "pve"
    
    pct_vmid: 105
    pct_hostname: "container-01"
    pct_password: 'securepassword123'
    pct_ostemplate: 'local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst'
  roles:
    - s4int.proxmox_ct
```

## Advanced Configuration

### Custom Network Configuration

You can define network settings using the raw `pct_netif` variable:

```yaml
pct_netif:
  net0: "name=eth0,ip=dhcp,ip6=dhcp,bridge=vmbr0,firewall=0"
```

Alternatively, you can use the more structured `pct_netif_dict` variable, which is automatically compiled into the format required by Proxmox. This avoids writing comma-separated strings manually:

```yaml
pct_netif_dict:
  net0:
    name: eth0
    bridge: vmbr0
    ip: dhcp
    firewall: 0
  net1:
    name: eth1
    bridge: vmbr1
    ip: 192.168.10.15/24
    gw: 192.168.10.1
    firewall: 1
```

Supported keys under each interface in `pct_netif_dict` include:

- `name` (the interface name inside the container, e.g., `eth0`)
- `bridge` (the Proxmox bridge to attach the interface to, e.g., `vmbr0`)
- `ip` (IPv4 address in CIDR format or `dhcp`)
- `gw` (default gateway for IPv4)
- `ip6` (IPv6 address in CIDR format or `dhcp`)
- `gw6` (default gateway for IPv6)
- `firewall` (enable/disable firewall rules, `1` or `0`)
- `link_down` (whether the link is down/plug pulled, `1` or `0`)
- `type` (interface type, e.g., `veth`)
- `hwaddr` (MAC address)
- `mtu` (MTU size)
- `rate` (rate limit in MB/s)
- `tag` (VLAN tag)
- `host-managed` (manage IP configuration via host)

### Custom Container Features

Set container features:

```yaml
pct_features:
  - nesting=1
  - keyctl=1
```

### Additional Mount Volumes

```yaml
pct_mount_volumes:
  - id: mp0
    mountpoint: /mnt/data
    storage: local-lvm
    size: 10
```

Mount directory from host to container:

```yaml
pct_mount_volumes:
  - id: mp0
    mountpoint: /mnt/data
    host_path: /mnt/data
```

## Usage

1. Ensure your Proxmox server is accessible and you have API credentials
2. Update inventory with your Proxmox server details and container configuration
3. Run the playbook

```shell
ansible-playbook -i inventory/hosts.yml playbook.yml
```

## License

MIT

## Author Information

Created for s4int infrastructure automation.
