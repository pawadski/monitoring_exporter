# monitoring_exporter

System metrics exporter for Prometheus. Written in Bash and served via Xinetd (and a small wrapper script that turns content HTTP-compliant).

## Installation

Tested on:
- Debian 7, 8, 9
- Ubuntu 14, 16, 18
- CentOS 5 (with a workaround), 6, 7
- CloudLinux 6, 7

Requires Bash >4, on CentOS 5 the playbook will attempt to compile a separate Bash version during install.

###  Ansible

Playbooks are included in playbooks/. 

You will likely need to edit the IP address variable `ip_address` - this is the IP address the playbook will add to the appropriate xinetd.d files, as a makeshift whitelist if you have only one Prometheus instance that's going to be contacting the exporter.

### Installing without Ansible

- Install Xinetd
- Place desired xinetd jobs from xinetd.d into /etc/xinetd.d/
- mkdir -p /opt/metrics.d/
- Place exporter in /opt/metrics.d/exporter_monitoring
- Place httpwrapper in /opt/metrics.d/
- chmod +x /opt/metrics.d/*
- Restart Xinetd

## No docker?!

This exporter is designed to run directly on the server. If you do make it work in Docker do let me know, though.
