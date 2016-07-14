openvpn
=======

Ansible role which helps to install and configure OpenVPN server.

The configuration of the role is done in such way that it should not be
necessary to change the role for any kind of configuration. All can be
done either by changing role parameters or by declaring completely new
configuration as a variable. That makes this role absolutely
universal. See the examples below for more details.

Please report any issues or send PR.


Examples
--------

This role requires the user to create the server side certificates. This can be
achieved with the [`openvpn_pki`](https://github.com/jtyr/openvpn_pki) make
file. The creation of the server certificates is as simple as this:

```
$ git clone https://github.com/jtyr/openvpn_pki.git
$ cd openvpn_pki
$ make server SERVER="vpn.server.com"
```

The server certificates are then used with the Ansible role as follows:

```
---

- name: Default installation
  hosts: all
  vars:
    # List of server-side certificates
    openvpn_ca_crt_source: /path/to/the/openvpn_pki/keys/ca.crt
    openvpn_server_key_source: /path/to/the/openvpn_pki/keys/vpn.server.com.key
    openvpn_server_crt_source: /path/to/the/openvpn_pki/keys/vpn.server.com.crt
    openvpn_dh_source: /path/to/the/openvpn_pki/keys/dh2048.pem
    openvpn_ta_source: /path/to/the/openvpn_pki/keys/ta.key
    openvpn_crl_source: /path/to/the/openvpn_pki/keys/crl.pem
  roles:
    - openvpn

- name: Example of the config file customization
  hosts: all
  vars:
    # List of server-side certificates
    openvpn_ca_crt_source: /path/to/the/openvpn_pki/keys/ca.crt
    openvpn_server_key_source: /path/to/the/openvpn_pki/keys/vpn.server.com.key
    openvpn_server_crt_source: /path/to/the/openvpn_pki/keys/vpn.server.com.crt
    openvpn_dh_source: /path/to/the/openvpn_pki/keys/dh2048.pem
    openvpn_ta_source: /path/to/the/openvpn_pki/keys/ta.key
    # Change default port number from 1194 to 5000
    openvpn_config_net_port: 5000
    # Suppress key persistency during the SIGUSR1 kill signal (set `null` value)
    openvpn_config_service_persist_key: null
    # Add support for Certificate Revocation List
    # (use `make revoke_gen_crl` or `make revoke CLIENT="client01"` to create it)
    openvpn_certs__custom:
      - src: /path/to/the/openvpn_pki/keys/crl.pem
        dest: "{{ openvpn_cert_dir }}/crl.pem"
    openvpn_config_certs__custom:
      crl-verify: "{{ openvpn_cert_dir }}/crl.pem"
  roles:
    - openvpn
```

The `server.conf` created by the first example looks like this:

```
ca /etc/openvpn/certs/ca.crt
cert /etc/openvpn/certs/server.crt
cipher AES-256-CBC
client-to-client #
comp-lzo #
dev tun
dh /etc/openvpn/certs/dh.pem
group openvpn
ifconfig-pool-persist ipp.txt
keepalive 1800 3600
key /etc/openvpn/certs/server.key
log /var/log/openvpn/server.log
max-clients 10
mute 20
persist-key #
persist-tun #
port 1194
proto tcp
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
server 10.8.0.128 255.255.255.128
status /var/log/openvpn/status.log
tls-auth /etc/openvpn/certs/ta.key 0
user openvpn
verb 3
```

The client side requires only the `.p12` and the `ta.key` file. Android and iOS
devices do not expose the CA certificate from the `.p12` file so it's necessary
to add extra `ca` parameter into the client configuration file and import the
`ca.crt` file to the device. The configuration file (`client01.ovpn`) can
look like this:

```
# Uncomment the following line for Android and iOS devices
#ca ca.crt
cipher AES-256-CBC
client
comp-lzo
dev tun
persist-key
persist-tun
pkcs12 clien01.p12
proto tcp
remote vpn.server.com 1194
tls-auth ta.key 1
verb 0
verify-x509-name "vpn.server.com" name
```


Role variables
--------------

Variables used by the role is as follows:

```
# Package to be installed (explicit version can be specified here)
openvpn_pkg: openvpn

# Whether to install EPEL YUM repo
openvpn_epel_install: no

# EPEL YUM repo URL
openvpn_epel_yumrepo_url: "{{ yumrepo_epel_url | default('https://dl.fedoraproject.org/pub/epel/$releasever/$basearch/') }}"

# Additional YUM repo params
openvpn_epel_yumrepo_params: {}

# Name of the config file which will be used to enable/start the service
openvpn_config_file: server

# Service name
# (Change the value to "openvpn@{{ openvpn_config_file }}" if using systemd)
openvpn_service: openvpn

# User and group
openvpn_user: openvpn
openvpn_group: openvpn

# Configuration directory
openvpn_config_dir: /etc/openvpn

# Directory for certificates
openvpn_cert_dir: "{{ openvpn_config_dir }}/certs"

# Directory for logs
openvpn_log_dir: /var/log/openvpn

# Log files to touch
openvpn_log_files:
  - "{{ openvpn_config_logs_server }}"
  - "{{ openvpn_config_logs_status }}"

# Paths to all the server keys and certificates (must be set as variable by the user!)
openvpn_ca_crt_source: ""
openvpn_server_key_source: ""
openvpn_server_crt_source: ""
openvpn_dh_source: ""
openvpn_ta_source: ""

# Path where all the server keys and certificates will be stored on the server
openvpn_ca_crt_dest: "{{ openvpn_cert_dir }}/ca.crt"
openvpn_server_key_dest: "{{ openvpn_cert_dir }}/server.key"
openvpn_server_crt_dest: "{{ openvpn_cert_dir }}/server.crt"
openvpn_dh_dest: "{{ openvpn_cert_dir }}/dh.pem"
openvpn_ta_dest: "{{ openvpn_cert_dir }}/ta.key"

# Default list of certificates to be copied to the remote host
openvpn_certs__default:
  - src: "{{ openvpn_ca_crt_source }}"
    dest: "{{ openvpn_ca_crt_dest }}"
  - src: "{{ openvpn_server_key_source }}"
    dest: "{{ openvpn_server_key_dest }}"
  - src: "{{ openvpn_server_crt_source }}"
    dest: "{{ openvpn_server_crt_dest }}"
  - src: "{{ openvpn_dh_source }}"
    dest: "{{ openvpn_dh_dest }}"
  - src: "{{ openvpn_ta_source }}"
    dest: "{{ openvpn_ta_dest }}"

# Custom list of certificates to be copied to the remote host
openvpn_certs__custom: []

# Final list of certificates to be copied to the remote host
openvpn_certs: "{{
  openvpn_certs__default +
  openvpn_certs__custom
}}"

# Values of the default certs part of the config file
openvpn_config_certs_ca: "{{ openvpn_ca_crt_dest }}"
openvpn_config_certs_key: "{{ openvpn_server_key_dest }}"
openvpn_config_certs_cert: "{{ openvpn_server_crt_dest }}"
openvpn_config_certs_dh: "{{ openvpn_dh_dest }}"
openvpn_config_certs_tls_auth: "{{ openvpn_ta_dest }} 0"

# Default certs part of the config file
openvpn_config_certs__default:
  ca: "{{ openvpn_config_certs_ca }}"
  key: "{{ openvpn_config_certs_key }}"
  cert: "{{ openvpn_config_certs_cert }}"
  dh: "{{ openvpn_config_certs_dh }}"
  tls-auth: "{{ openvpn_config_certs_tls_auth }}"

# Custom certs part of the config file
openvpn_config_certs__custom: {}

# Certs part of the config file
openvpn_config_certs: "{{
  openvpn_config_certs__default.update(openvpn_config_certs__custom) }}{{
  openvpn_config_certs__default }}"


# Values of the default networking part of the config file
openvpn_config_net_dev: tun
openvpn_config_net_proto: tcp
openvpn_config_net_port: 1194
openvpn_config_net_server_network: 10.8.0.128
openvpn_config_net_server_mask: 255.255.255.128
openvpn_config_net_server: "{{ openvpn_config_net_server_network }} {{ openvpn_config_net_server_mask }}"
openvpn_config_net_ifconfig_pool_persist: ipp.txt
openvpn_config_net_push_other: []
openvpn_config_net_push_dns:
  - '"dhcp-option DNS 8.8.8.8"'
  - '"dhcp-option DNS 8.8.4.4"'
openvpn_config_net_push: "{{
  openvpn_config_net_push_other +
  openvpn_config_net_push_dns
}}"

# Default networking part of the config file
openvpn_config_net__default:
  dev: "{{ openvpn_config_net_dev }}"
  proto: "{{ openvpn_config_net_proto }}"
  port: "{{ openvpn_config_net_port }}"
  server: "{{ openvpn_config_net_server }}"
  ifconfig-pool-persist: "{{ openvpn_config_net_ifconfig_pool_persist }}"
  push: "{{ openvpn_config_net_push }}"

# Custom networking part of the config file
openvpn_config_net__custom: {}

# Networking part of the config file
openvpn_config_net: "{{
  openvpn_config_net__default.update(openvpn_config_net__custom) }}{{
  openvpn_config_net__default }}"


# Values of the default connection part of the config file
openvpn_config_con_client_to_client: "#"
openvpn_config_con_keepalive_ping: 1800
openvpn_config_con_keepalive_restart: 3600
openvpn_config_con_keepalive: "{{ openvpn_config_con_keepalive_ping }} {{ openvpn_config_con_keepalive_restart }}"
openvpn_config_con_cipher: AES-256-CBC
openvpn_config_con_comp_lzo: "#"
openvpn_config_con_max_clients: 10

# Default connection part of the config file
openvpn_config_con__default:
  client-to-client: "{{ openvpn_config_con_client_to_client }}"
  keepalive: "{{ openvpn_config_con_keepalive }}"
  cipher: "{{ openvpn_config_con_cipher }}"
  comp-lzo: "{{ openvpn_config_con_comp_lzo }}"
  max-clients: "{{ openvpn_config_con_max_clients }}"

# Custom connection part of the config file
openvpn_config_con__custom: {}

# Connection part of the config file
openvpn_config_con: "{{
  openvpn_config_con__default.update(openvpn_config_con__custom) }}{{
  openvpn_config_con__default }}"


# Values of the default service part of the config file
openvpn_config_service_persist_key: "#"
openvpn_config_service_persist_tun: "#"

# Default connection service of the config file
openvpn_config_service__default:
  persist-key: "{{ openvpn_config_service_persist_key }}"
  persist-tun: "{{ openvpn_config_service_persist_tun }}"

# Custom connection part of the config file
openvpn_config_service__custom: {}

# Connection part of the config file
openvpn_config_service: "{{
  openvpn_config_service__default.update(openvpn_config_service__custom) }}{{
  openvpn_config_service__default }}"


# Values of the default logs part of the config file
openvpn_config_logs_server: "{{ openvpn_log_dir }}/server.log"
openvpn_config_logs_status: "{{ openvpn_log_dir }}/status.log"
openvpn_config_logs_verb: 3
openvpn_config_logs_mute: 20

# Default logs service of the config file
openvpn_config_logs__default:
  log: "{{ openvpn_config_logs_server }}"
  status: "{{ openvpn_config_logs_status }}"
  verb: "{{ openvpn_config_logs_verb }}"
  mute: "{{ openvpn_config_logs_mute }}"

# Custom logs part of the config file
openvpn_config_logs__custom: {}

# Logs part of the config file
openvpn_config_logs: "{{
  openvpn_config_logs__default.update(openvpn_config_logs__custom) }}{{
  openvpn_config_logs__default }}"


# Values of the default user part of the config file
openvpn_config_user_user: "{{ openvpn_user }}"
openvpn_config_user_group: "{{ openvpn_group }}"

# Default logs user of the config file
openvpn_config_user__default:
  user: "{{ openvpn_config_user_user }}"
  group: "{{ openvpn_config_user_group }}"

# Custom user part of the config file
openvpn_config_user__custom: {}

# User part of the config file
openvpn_config_user: "{{
  openvpn_config_user__default.update(openvpn_config_user__custom) }}{{
  openvpn_config_user__default }}"


openvpn_config_tmp: {}

# Final configuration
openvpn_config: "{{
  openvpn_config_tmp.update(openvpn_config_certs) }}{{
  openvpn_config_tmp.update(openvpn_config_net) }}{{
  openvpn_config_tmp.update(openvpn_config_con) }}{{
  openvpn_config_tmp.update(openvpn_config_service) }}{{
  openvpn_config_tmp.update(openvpn_config_logs) }}{{
  openvpn_config_tmp.update(openvpn_config_user) }}{{
  openvpn_config_tmp }}"
```


Dependencies
------------

- [`config_encoder_filters`](https://github.com/jtyr/config_encoder_filters)
- [`openvpn_pki`](https://github.com/jtyr/openvpn_pki) (optional)


License
-------

MIT


Author
------

Jiri Tyr
