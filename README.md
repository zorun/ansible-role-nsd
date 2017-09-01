# Ansible role for NSD

It is assumed that all machines host the exact same zones in the same way
(i.e. for each zone, they are either all masters, or all slaves).
This greatly simplifies configuration, and there is generally no need
to have masters and slaves for the same zone handled by Ansible, since
Ansible can just push the zone to all machines (so, they can all be masters).

There are cases where having multiple masters whose zone is pushed by Ansible
is not desirable:

- dynamic DNS records (which isn't supported by NSD anyway)
- DNSSEC (you could always generate DNSSEC signatures in ansible, but it's awkward)


You can put key-value pairs in group_vars, under the key "nsd_common_config",
they will be used for the NSD configuration on all hosts.
A few common configuration entries are in the playbook itself, because they are
needed to properly create relevant directories.

Some machine-specific configuration is also possible, e.g. for the bind IP.
Put key-value pairs in host_vars, under the key "nsd_local_config".

If you want to pass multiple values for a key (e.g. ip-address), just use
a list as value, it will automatically be expanded.

The zones are configured in group_vars, see the example.  It is possible
to optionally add a TSIG key to each slave/master, see again the example.
The zone files themselves for primary zones should be put in files/nsd.

The playbook is currently only tested with Debian wheezy.

When a master zone is updated, the slaves are notified.  Obviously, the slaves
need to be configured to accept notification from at least one master and pull
zones accordingly.

## License

MIT

