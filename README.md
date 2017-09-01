# Ansible role for NSD

This Ansible role installs and configure NSD, an authoritative DNS server.

## Features

This role supports both NSD3 and NSD4, allows to manage both NSD configuration
and zone files, in master or slave mode, with or without zone transfer (TSIG).

Most notably, compared to other NSD roles ([elnappo](https://github.com/elnappo/ansible-role-nsd),
[reallyenglish](https://github.com/reallyenglish/ansible-role-nsd), [creicm](https://github.com/criecm/ansible-role-nsd),
[adarnimrod](https://github.com/adarnimrod/nsd), [hudecof](https://github.com/hudecof/ansible_authoritative_dns)),
it has the following features:

- allows to store zone data in "classical" zone files, instead of having to write
  zones as ansible variables like [here](https://github.com/reallyenglish/ansible-role-nsd)
- supports both master and slave scenarios, or even a mixture of both
  (some zones running as master and some zones running as slave, on the same
  NSD server)
- completely generic NSD configuration (you are free to define any configuration
  attribute supported by NSD, there is no hard-coded list in the role).  This is
  similar to [here](https://github.com/reallyenglish/ansible-role-nsd) but with a
  simpler syntax (automatic expansion of lists)
- supports zone transfers: TSIG keys, notify slaves...
- hopefully flexible yet simple to use!

However, it does not handles other things like firewall configuration or emails,
and only supports Debian for now.

## Use-cases

Note: all these use-cases can be used independently for each zone!  That is,
a single NSD server can be both a master for `example.com` and a slave for
`example.org`.

### Pure multi-master

If all your DNS servers are configured with Ansible, this is the simplest setup:
all servers are masters for the zone, and zone data is deployed using Ansible.

In this setup, there is no need for DNS-based zone transfers.

### Multi-master with additional slaves not configured by this role

In this setup, one or more master is configured using Ansible, like in the previous
use-case.  However, DNS-based zone transfer is also setup, to allow external slaves
to fetch zone data from a master.  Whenever Ansible updates a zone on the masters,
it will tell NSD to notify the slaves.

Note that in this setup, it is your responsibility to configure the slaves appropriately
(accept notify from all masters, and fetch zone data from one or more masters).

### Slave only

In this setup, the NSD servers are simply configured as slaves for the zone.
In this case, no zone data is configured on the servers, because data will be fetched
from an external master using normal zone transfer mechanisms.

Note that in this setup, it is your responsibility to configure the master appropriately
(notify the slaves, and allow zone transfers from the slaves).

## Requirements

This role has been tested on Debian (wheezy, jessie, stretch), but could be adapted
to work on other systems.  Notably, this role does not setup `nsd-control` because
this is already done automatically by the Debian package.

By default, NSD4 is assumed, you will need additional configuration for NSD3 (wheezy).

## Role variables

This documents all role variables that you can set in your playbooks.  See the end
of this README for a complete example.

### Server configuration

    nsd_server_config

Dictionary of key-value pairs that will be added to the `server:` configuration
section of NSD.  A value can be either a string or a list.  In the latter case,
the value will be expanded into multiple configuration entries.

You can add any configuration option you want in this dictionary, but make sure
that it is a configuration option understood by NSD!
As a safety net, the role asks NSD to verify the generated configuration before
proceeding, but this will not be done when running `ansible-playbook --check`.


    nsd_local_server_config

Same syntax and semantic as `nsd_server_config`.
This second variable is provided so that it's easier to add specific configuration
to a single machine.  You would typically define `nsd_server_config` in `group_vars`
or in a playbook, while `nsd_local_server_config` would be defined in `host_vars`.

### TSIG keys

    nsd_tsig_keys

List of TSIG keys.  Each key must be a dictionary with the following attributes:

  * `tsig_keyname`: name of this TSIG key.  Note that in a master/slave DNS configuration,
    the name of the TSIG key must be the same on master and slaves!  This name is also
    used to refer to TSIG keys from the `nsd_primary_zones` and `nsd_secondary_zones` role
    variables.  **Mandatory**.
  * `tsig_secret`: base64-encoded value of the key.  **Mandatory**.
  * `tsig_algorithm`: algorithm used by the key, for instance `hmac-md5`.  **Mandatory**.

### Primary zones

    nsd_primary_zones

List of zones to be served as master.  Each zone must be a dictionary with the following
attributes:

  * `zone_name`: name of the zone, for instance `example.com` or `8.b.d.0.1.0.0.2.ip6.arpa.`.  **Mandatory**.
  * `zone_filename`: name of the file containing zone data (will be searched for in `files/nsd/`).  **Mandatory**.
  * `slaves`: list of DNS slaves, format is described below.  **Optional**.

The format for a slave entry is the following:

  * `ip`: IPv4 or IPv6 address of the DNS slave.  Will be used to send "notify" messages,
    and to allow zone transfers from this IP.  **Mandatory**.
  * `tsig_key`: name of the TSIG key to use when communicating with this slave.  The name
    must match the `tsig_keyname` field of a previously-defined TSIG key, see above.  **Optional**.

### Secondary zones

    nsd_secondary_zones

List of zones to be served as slave.  Each zone must be a dictionary with the following
attributes:

  * `zone_name`: name of the zone, for instance `example.com` or `8.b.d.0.1.0.0.2.ip6.arpa.`.  **Mandatory**.
  * `masters`: list of DNS masters, format is described below.  **Optional** but a slave zone is quite useless without master.

The format for a master entry is the following:

  * `ip`: IPv4 or IPv6 address of the DNS master.  Will be used to request zone transfers,
    and to allow notify messages.  **Mandatory**.
  * `tsig_key`: name of the TSIG key to use when communicating with this master.  The name
    must match the `tsig_keyname` field of a previously-defined TSIG key, see above.  **Optional**.

### Advanced configuration variables

These variables should not need to be changed in most cases.
Variables are presented here with their default value.

    nsd_version: 4

The version of NSD.  Used to skip tasks or handlers that do not make
sense depending on the version.

    nsd_service_name: "nsd"

The name of the init service, used to restart NSD.

    nsd_pkg_name: "nsd"

Name of the Debian package to install.

    nsd_control_program: "/usr/sbin/nsd-control"

Program used to control NSD, for reload, rebuild, notify...

    nsd_config_dir: "/etc/nsd"

Directory where NSD configuration will be stored.

    nsd_zones_config_file: "/etc/nsd/zones.conf"

Name of the config file that will hold zone configuration (it is then
included from the main NSD configuration file).

    nsd_primary_zones_dir: "/etc/nsd/primary"

Directory where the zone files will be copied by this role.

    nsd_secondary_zones_dir: "/etc/nsd/secondary"

Directory where slave zone files will be placed by NSD after zone transfer.

### Configuration for NSD3

For NSD3 on Debian an appropriate configuration would be:

    nsd_version: 3
    nsd_service_name: "nsd3"
    nsd_pkg_name: "nsd3"
    nsd_control_program: "/usr/sbin/nsdc"
    nsd_config_dir: "/etc/nsd3"
    nsd_zones_config_file: "/etc/nsd3/zones.conf"
    nsd_primary_zones_dir: "/etc/nsd3/primary"
    nsd_secondary_zones_dir: "/etc/nsd3/secondary"

This could be placed in a `group_vars/wheezy.yml` file or equivalent.

## Example playbook

This is a complete example playbook with several TSIG keys and several DNS zones:
the first zone is a primary zone with no slave, the second zone has two slaves, and
the third zone is a secondary zone with two masters.

    - hosts: dnsservers
      roles:
        - nsd
      vars:
        nsd_server_config:
          verbosity: 2
          ip4-only: 'yes'
        nsd_tsig_keys:
        # Two TSIG keys, used in zone definition below
          - tsig_keyname: "tsig-key.example.org"
            tsig_secret: "3znH//y866vzpOZdahYYUlWeiY4iidiJGFRX6CI6OkUBggRNYFpZAMvlYbtnUosiBVPsgghA6zT0TzOEX0vetQ=="
            tsig_algorithm: hmac-md5
          - tsig_keyname: "key-eu.org"
            tsig_secret: "t6ELXqsSLYl57iO2rxj+X9+DNpOV3exTBFWu9wS/3jI="
            tsig_algorithm: hmac-sha256
        nsd_primary_zones:
        # Master zone without slave
          - zone_name: "example.com."
            zone_filename: "example.com.zone"
        # Master zone with two slaves, one dual-stacked with a TSIG key, and one single-stacked with no key
          - zone_name: "example.org."
            zone_filename: "example.org.zone"
            slaves:
              - ip: 2001:db8:42:1337::1
                tsig_key: "tsig-key.example.org"
              - ip: 198.51.100.12
                tsig_key: "tsig-key.example.org"
              - ip: 203.0.113.8
        nsd_secondary_zones:
        # Slave zone with two masters, first one without key, second one with a key
          - zone_name: "example.eu.org"
            masters:
              - ip: 192.0.2.42
              - ip: 2001:db8:1234:5678::9
                tsig_key: "key-eu.org"

Also in `host_vars/ns1.yml`:

    nsd_local_server_config:
      ip-address: ['2001:db8:ffff::42', '203.0.113.199']

The zone data for the two primary zones needs to be stored in `files/nsd/` at
the root of your ansible directory:

    # ls files/nsd/
    example.org.zone   example.com.zone
    # head -3 files/nsd/example.org.zone
    $ORIGIN example.org
    $TTL 3h
    @  IN  SOA  ns1 root.example.org. (2017090101 1d 2h 4w 1h)


## Limitations

For a given zone, all machines using this role must be either all masters or all slaves.

This greatly simplifies configuration, and there is generally no need
to have masters and slaves for the same zone handled by Ansible, since
Ansible can just push the zone data to all servers (so, they can all be masters).

There are cases where having multiple masters whose zone is pushed by Ansible
would probably not be desirable:

- dynamic DNS records (which isn't supported by NSD anyway)
- DNSSEC (you could always generate DNSSEC signatures in ansible, but it's awkward)

Thus, these use-cases are not supported.

Of course, you can workaround this limitation by simply calling this role multiple
times with different machines and different configuration.

## License

MIT

