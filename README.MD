# ansible-ceph_common

Role for deploy:

* `ceph` packages
* `ceph.conf` configuration
* `cephx` auth keys
* `radosgw` rsa certificate bundles
* `rbdmap` spec files

## Requirements

* Ansible 3.0.0+

Example configuration
-------------------------

```yaml
---
ceph_common:
# Install/upgrade ceph packages or not
  - install_package: 'true'
# 'present' (do nothing if package is already installed) or 'latest' (always
# upgrade to last version)
    package_state: 'present'
# Enable rbdmap service or not
    rbdmap_enable: 'false'
    settings:
# [global] in ceph.conf, all options passed as-is without any validation
      - global:
          - mon_host: '10.10.10.2:3300,10.10.10.3:3300,10.10.10.4:3300'
            public_network: '10.10.10.0/24'
            fsid: '8ca762db-c2a8-4008-a7f1-750be9e3eaaf'
# "global" [client] in ceph.conf
        client:
# [client.<name>] in ceph.conf
          - name: 'rgw_client'
            settings:
              - host: 'rgw_client'
                log_file: '""'
                log_to_syslog: 'true'
                rgw_enable_lc_threads: 'false'
                rgw_enable_gc_threads: 'false'
                debug_rgw: '0'
                debug_beast: '0'
                rgw_frontends: 'beast endpoint=127.0.0.1:8087'
# keyrings for cephx
    keyrings:
      - name: 'john'
        type: 'client'
        key: 'AQA2xWFa/FtQOBAAmPxw67jN6YocI7B18XwciA=='
      - name: 'admin'
        type: 'client'
        key: '<key>'
      - name: 'ceph'
        type: 'bootstrap-mgr'
        key: '<key>'
      - name: "{{ inventory_hostname_short }}"
        type: 'mon'
        key: '<key>'
      - name: "{{ inventory_hostname_short }}"
        type: 'mgr'
        key: "{{ ceph_common_keyrings_mgr_keyring | default(omit) }}"
# RBD spec file for rbdmap service
    rbdmap:
      - pool: 'rbd'
        image: 'volume-f070e1b3-f745-45be-8653-ac16373fa60d'
        user: 'john'
        keyring: 'john'
# Most of these options are useful mainly for debugging and benchmarking. The
# default values are set in the kernel and may therefore depend on the version
# of the running kernel
        options:
# FSID that should be assumed by the client
          - fsid: 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee'
# IP and, optionally, port the client should use
            ip: 'a.b.c.d[:p]'
# Enable sharing of client instances with other mappings (default)
            share: 'true'
# Disable sharing of client instances with other mappings
            noshare: 'false'
# Enable CRC32C checksumming for msgr1 on-the-wire protocol (default).
# For msgr2.1 protocol this option is ignored: full checksumming is always on in
# 'crc' mode and always off in 'secure' mode
            crc: 'true'
# Disable CRC32C checksumming for msgr1 on-the-wire protocol. Note that only
# payload checksumming is disabled, header checksumming is always on. For
# msgr2.1 protocol this option is ignored
            nocrc: 'false'
# Disable Nagle's algorithm on client sockets (since 4.0, default)
            tcp_nodelay: 'true'
# Enable Nagle's algorithm on client sockets (since 4.0)
            notcp_nodelay: 'fasle'
# A timeout on various steps in rbd device map and rbd device unmap sequences
# (default is 60 seconds). In particular, since 4.2 this can be used to ensure
# that rbd device unmap eventually times out when there is no network connection
# to a cluster
            mount_timeout: '60'
# OSD keepalive timeout (default is 5 seconds)
            osdkeepalive: '5'
# OSD idle TTL (default is 60 seconds)
            osd_idle_ttl: '60'
# Map the image read-write (default)
            rw: 'true'
# Map the image read-only
            ro: 'false'
# Queue depth (since 4.2, default is 128 requests)
            queue_depth: '128'
# Acquire exclusive lock on reads, in addition to writes and discards
# (since 4.9)
            lock_on_read: 'false'
# Disable automatic exclusive lock transitions (since 4.12)
            exclusive: 'false'
# A timeout on waiting for the acquisition of exclusive lock (since 4.17,
# default is 0 seconds, meaning no timeout)
            lock_timeout: '0'
# Turn off discard and write zeroes offload support to avoid deprovisioning a
# fully provisioned image (since 4.17). When enabled, discard requests will fail
# with -EOPNOTSUPP, write zeroes requests will fall back to manually zeroing.
            notrim: 'true'
# Fail write requests with -ENOSPC when the cluster is full or the data pool
# reaches its quota (since 5.0). The default behaviour is to block until the
# full condition is cleared.
            abort_on_full: 'false'
# Minimum allocation unit of the underlying OSD object store backend (since 5.1,
# default is 64K bytes). This is used to round off and drop discards that are
# too small. For bluestore, the recommended setting is bluestore_min_alloc_size
# (typically 64K for hard disk drives and 16K for solid-state drives). For
# filestore with filestore_punch_hole = false, the recommended setting is image
# object size (typically 4M).
            alloc_size: ''
# Specify the location of the client in terms of CRUSH hierarchy (since 5.8).
# This is a set of key-value pairs separated from each other by '|', with keys
# separated from values by ':'. Note that '|' may need to be quoted or escaped
# to avoid it being interpreted as a pipe by the shell. The key is the bucket
# type name (e.g. rack, datacenter or region with default bucket types) and the
# value is the bucket name. For example, to indicate that the client is local
# to rack "myrack", data center "mydc" and region "myregion":
# crush_location: 'rack:myrack\|datacenter:mydc\|region:myregion'
# Each key-value pair stands on its own: "myrack"doesn't need to reside in
# "mydc", which in turn doesn't need to reside in "myregion". The location is
# not a path to the root of the hierarchy but rather a set of nodes that are
# matched independently, owning to the fact that bucket names are unique within
# a CRUSH map. "Multipath" locations are supported, so it is possible to
# indicate locality for multiple parallel hierarchies:
                  crush_location: 'rack:myrack1\|rack:myrack2\|datacenter:mydc'
# 'no' - disable replica reads, always pick the primary OSD (since 5.8,
# default)
# 'balance' - when issued a read on a replicated pool, pick a random OSD for
# serving it (since 5.8). This mode is safe for general use only since Octopus
# (i.e. after "ceph osd require-osd-release octopus"). Otherwise it should
# be limited to read-only workloads such as images mapped read-only everywhere
# or snapshots
# 'localize' - when issued a read on a replicated pool, pick the most local
# OSD for serving it (since 5.8). The locality metric is calculated against the
# location of the client given with 'crush_location', a match with the
# lowest-valued bucket type wins. For example, with default bucket types, an
# OSD in a matching rack is closer than an OSD in a matching data center, which
# in turn is closer than an OSD in a matching region. This mode is safe for
# general use only since Octopus (i.e. after "ceph osd require-osd-release
# octopus"). Otherwise it should be limited to read-only workloads such as
# images mapped read-only everywhere or snapshots
            read_from_replica: 'no'
# 'none' - on't set compression hints (since 5.8, default)
# 'compressible' - hint to the underlying OSD object store backend that the
# data is compressible, enabling compression in passive mode (since 5.8)
# 'incompressible' - hint to the underlying OSD object store backend that the
# data is incompressible, disabling compression in aggressive mode (since 5.8)
            compression_hint: 'none'
# 'legacy' - se msgr1 on-the-wire protocol (since 5.11, default)
# 'crc' - use msgr2.1 on-the-wire protocol, select 'crc' mode, also referred to
# as plain mode (since 5.11). If the daemon denies 'crc' mode, fail the
# connection
# 'secure' - use msgr2.1 on-the-wire protocol, select 'secure' mode
# (since 5.11). 'secure' mode provides full in-transit encryption ensuring both
# confidentiality and authenticity. If the daemon denies 'secure' mode, fail
# the connection
# 'prefer-crc' - use msgr2.1 on-the-wire protocol, select 'crc' mode (since
# 5.11). If the daemon denies 'crc' mode in favor of 'secure' mode, agree to
# 'secure' mode
# 'prefer-secure' - use msgr2.1 on-the-wire protocol, select 'secure' mode
# (since 5.11). If the daemon denies 'secure' mode in favor of 'crc' mode, agree
# to 'crc' mode
            ms_mode: 'prefer-crc'
# Wait for udev device manager to finish executing all matching "add" rules and
# release the device before exiting (default). This option is not passed to the
# kernel
            udev: 'true'
# Don't wait for udev device manager. When enabled, the device may not be fully
# usable immediately on exit
            noudev: 'false'
```
