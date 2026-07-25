# Ansible Role: StrongSwan

Ansible role for installing and configuring strongSwan IPsec VPN with the modern swanctl (VICI) interface.

This role is designed for site-to-site VPN setups with certificate-based authentication (pubkey). It uses a flexible, Jinja2-templated configuration that can be easily adapted to your inventory.

Based on [serverbee/ansible-role-strongswan](https://github.com/serverbee/ansible-role-strongswan)

## Features

- ✅ Installs strongSwan and necessary plugins (standard + extra).
- ✅ Uses the modern swanctl (VICI) interface.
- ✅ Optimized plugin loading for minimal footprint and maximum performance.
- ✅ Flexible connection definitions via a list of dictionaries, each rendered as a separate .conf file in /etc/swanctl/conf.d/.
- ✅ Connection defaults to avoid repetition and ensure consistency.
- ✅ Any plugin can be configured via the strongswan_plugins_config variable.
- ✅ Automatically disables and masks the legacy strongswan-starter service to prevent conflicts.
- ✅ Supports Let's Encrypt certificates for the server.
- ✅ Clean, modern Ansible syntax.

## Requirements

- Ansible Core >= 2.17
- Target system: Debian 12/13 or Ubuntu 24.04 (LTS).
- Root access.

## Role Variables

### Main Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| strongswan_conn_list | List of connections. Each item requires a name (filename) and a connection dictionary. See example below. | [] |
| strongswan_connection_defaults | Default values for all connections (e.g., version, proposals). | See defaults/main.yml |
| strongswan_plugins_config | Any plugin can be configured here. Key is the plugin name, value is a dict of its options. | {} |
| strongswan_module_settings | Core strongSwan settings (plugin load lists, charon options). | Optimized for site-to-site. |
| strongswan_letsencrypt_enable | Fetch certificates from Let's Encrypt. | false |
| strongswan_disable_starter | Disable the legacy strongswan-starter service. | true |

### Connection Variables

These are the keys you can use within each connection dict in strongswan_conn_list.

| Key | Description | Required |
|-----|-------------|----------|
| local_addrs | Local IP address(es). | Yes |
| remote_addrs | Remote peer IP address(es). | Yes |
| local | Dict with auth, certs, id, etc. | Yes |
| remote | Dict with auth, id, etc. | Yes |
| children | Dict of Child Security Associations. | Yes |
| fragmentation | Enable IKE fragmentation. | No (default: yes) |
| mobike | Enable MOBIKE. | No (default: no) |
| proposals | IKE proposal string. | No |
| Any other valid swanctl connection option | E.g., dpd_delay, rekey_time. | No |

## Dependencies

None.

## Example Playbooks

### Basic Site-to-Site Server

```yaml 
- hosts: vpn-gw-01 
  vars: 
    strongswan_conn_list:
      - name: office-01
        connection: 
          local_addrs: "10.0.1.1" 
          remote_addrs: "10.0.2.1" 
          local: 
            auth: pubkey 
            certs: "{{ inventory_hostname }}.pem" 
            id: "vpn.office-a.example.com" 
          remote: 
            auth: pubkey 
            id: "vpn.office-b.example.com" 
            children: 
              office-net: 
                local_ts: "10.0.1.0/24" 
                remote_ts: "10.0.2.0/24" 
              start_action: start 
              updown: "/etc/swanctl/scripts/updown.sh" 
              if_id_out: "1" 
              if_id_in: "1" 
    strongswan_plugins_config: 
      bypass-lan: 
        interfaces_use: "{{ ansible_default_ipv4.interface }}"
  roles: 
  - nightsnake.strongswan
```

### Using Connection Defaults

```yaml
 # group_vars/all.yml
 strongswan_connection_defaults:
    version: 2
    fragmentation: yes
    mobike: no
    proposals: aes256gcm16-sha384-prfsha384-ecp384
    local: 
      auth: pubkey
    remote: 
      auth: pubkey
    children_defaults: 
      start_action: start
      esp_proposals: aes256gcm16-ecp384 

# host_vars/vpn-01.yml
 strongswan_conn_list:
 - name: peer-01
   connection: 
     local_addrs: "10.0.1.1"
     remote_addrs: "10.0.2.1"
     local:
       certs: "vpn-01.pem"
    # Only define what's different
       id: "vpn-01.example.com"
     remote:
       id: "peer-01.example.com"
     children:
       child-01:
         local_ts: "10.0.1.0/24"
         remote_ts: "10.0.2.0/24"
```

### Configuring a Custom Plugin

```yaml 
strongswan_plugins_config: 
  bypass-lan: 
    interfaces_use: "eth0" 
  attr: 
    virtual_ip: "10.0.0.0/24" 
  virtual_ip_pools: "10.0.0.1-10.0.0.100"
```

## Development & Testing

1. Clone the repository.
2. Install dependencies: ansible-galaxy install -r requirements.yml
3. Run linters: ansible-lint . and yamllint .

## License

MIT

## Author Information

nightsnake