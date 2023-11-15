# Monitoring Exporter

System metrics exporter for Prometheus. Written in Bash and served via Xinetd (and a small wrapper script that turns content HTTP-compliant). The exporter provides reasonably fast response times by offloading methods to subprocesses (see `workers=2`). However, the response time will always be at least 1 second, as the exporter calculates current CPU usage by measuring it over a second.

#### A note on monitoring CPU usage

This exporter provides you with two CPU usage metrics. One of them is taken from /proc/stat verbatim (`monitoring_stat_cpu_seconds_total`). The other is a computed CPU usage. The exporter computes CPU usage by measuring it over one second. **The CPU usage calculation ignores iowait - iowait is not CPU usage, the processor is happy to do other things while waiting on disk**.

## Table of Contents

1. [Installation](#installation)
2. [Grafana Dashboard](#grafana)
3. [Metrics and Configuration](#metrics-and-configuration)
4. [Extending](#extending)

## Installation

Tested on:
- Debian 7, 8, 9
- Ubuntu 14, 16, 18
- CentOS 5 (with a workaround), 6, 7
- CloudLinux 6, 7

Requires Bash >4, on CentOS 5 for example, you will need to install Bash 4 (perhaps into a separate binary ie. `/bin/bash42`) and edit `bashPath` to it in the exporter file. You may also need to change the shebang lines, depending on your system.

###  Ansible

Playbook is included in `monitoring_exporter.yml`.

You will likely need to edit the IP address variable `ip_address` - this is the IP address the playbook will add to the appropriate xinetd.d files, as a makeshift whitelist if you have only one Prometheus instance that's going to be contacting the exporter.   

You can run the playbook with variables using the `-e` parameter: `ansible-playbook -l web_servers -e ip_address=123.123.123.123 monitoring_exporter.yml`

Or fetching your public IP dynamically: `ansible-playbook -l web_servers -e ip_address=$(curl -s icanhazip.com) monitoring_exporter.yml`

### Installing without Ansible

- Install Xinetd
- "template" the `exporter_monitoring.xinetd` file into /etc/xinetd.d/ (you will need to change the IP address setting inside or remove it if you do not intend on using the Xinetd whitelist)
- `mkdir -p /opt/metrics.d/`
- Place `exporter_monitoring` in `/opt/metrics.d/exporter_monitoring`
- Place `httpwrapper` in `/opt/metrics.d/httpwrapper`
- `chmod +x /opt/metrics.d/*`
- Restart Xinetd

## Grafana

A Grafana dashboard can be found here: https://grafana.com/grafana/dashboards/12095

## No docker?!

This exporter is designed to run directly on the server. If you do make it work in Docker do let me know, though.

# Metrics and Configuration

## Port 10100

Provides the following metrics, separated by functions. All module exports are prefixed with `monitoring_` in code.   

## Configuring

You are free to select only metrics relevant to you by editing this line:   
`declare -a exporters=(loadavg stat iostat filesystem memory netdev netstat uptime kernel file_nr)`   

#### loadavg (default: enabled)

Exports data from `/proc/loadavg`

| name                               | description                                      | additional labels | units |
|------------------------------------|--------------------------------------------------|-------------------|-------|
| monitoring_loadavg_load1           | 1 minute load average, as seen in /proc/loadavg  | none              | N     |
| monitoring_loadavg_load5           | 5 minute load average, as seen in /proc/loadavg  | none              | N     |
| monitoring_loadavg_load15          | 15 minute load average, as seen in /proc/loadavg | none              | N     |
| monitoring_loadavg_jobs_running    | Number of running jobs                           | none              | N     |
| monitoring_loadavg_jobs_background | Number of background jobs                        | none              | N     |

#### stat (default: enabled)

Exports *some* data from `/proc/stat`

| name                                       | description                                                             | additional labels | units      |
|--------------------------------------------|-------------------------------------------------------------------------|-------------------|------------|
| monitoring_stat_cpu_seconds_total          | Total time in ticks (usually seconds) the CPU has spent in each mode    | mode              | Seconds    |
| monitoring_stat_cpu_current_usage_percent  | CPU usage at the moment of execution `1`                                  | none              | Percentage |
| monitoring_stat_cpu_current_iowait_percent | iowait at the moment of execution `2` - same you'd see when running "top" | none              | Percentage |

`1.` calculated by subtracting `%iowait`+`%idle` from 100 over 1 second: `100 - (idle + iowait)`   
`2.` calculated by subtracting `iowait` from `idle` over 1 second

#### filesystem (default: enabled)

Exports data from `df`

| name                                   | description                                  | additional labels              | units      |
|----------------------------------------|----------------------------------------------|--------------------------------|------------|
| monitoring_filesystem_total_kbytes     | Total amount of kbytes space in a filesystem | mountpoint, source, fstype `1` | kbytes     |
| monitoring_filesystem_used_kbytes      | Used kbytes in filesystem                    | mountpoint, source, fstype `1` | kbytes     |
| monitoring_filesystem_avail_kbytes     | Available kbytes in a filesystem             | mountpoint, source, fstype `1` | kbytes     |
| monitoring_filesystem_capacity_percent | Percentage usage of the filessystem          | mountpoint, source, fstype `1` | Percentage |

`1.` mountpoint: the mount point in local system   
`1.` source: the block device   
`1.` fstype: filesystem type

#### iostat (default: enabled)
Exports data from `iostat -xm`

| name                       | description                                                                                                                                                                                   | additional labels | units      |
|----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------|------------|
| monitoring_iostat_rrqm     | The number of read requests merged per second that were queued to the device                                                                                                                  | device*           | N          |
| monitoring_iostat_wrqm     | The number of write requests merged per second that were queued to the device                                                                                                                 | device*           | N          |
| monitoring_iostat_r        | The number (after merges) of read requests completed per second for the device                                                                                                                | device*           | N          |
| monitoring_iostat_w        | The number (after merges) of write requests completed per second for the device                                                                                                               | device*           | N          |
| monitoring_iostat_rmb      | The number of megabytes read from the device per second                                                                                                                                       | device*           | N          |
| monitoring_iostat_wmb      | The number of megabytes written to the device per second                                                                                                                                      | device*           | N          |
| monitoring_iostat_avgrq_sz | The average size (in sectors) of the requests that were issued to the device                                                                                                                  | device*           | N          |
| monitoring_iostat_avgqu_sz | The average queue length of the requests that were issued to the device                                                                                                                       | device*           | N          |
| monitoring_iostat_await    | The average time (in milliseconds) for I/O requests issued to the device to be served. This includes the time spent by the requests in queue and the time spent servicing them                | device*           | ms         |
| monitoring_iostat_r_await  | The average time (in milliseconds) for read requests issued to the device to be served. This includes the time spent by the requests in queue and the time spent servicing them               | device*           | ms         |
| monitoring_iostat_w_await  | The average time (in milliseconds) for write requests issued to the device to be served. This includes the time spent by the requests in queue and the time spent servicing them              | device*           | ms         |
| monitoring_iostat_svctm    | The average service time (in milliseconds) for I/O requests that were issued to the device. Warning! Do not trust this field any more. This field will be removed in a future sysstat version | device*           | ms         |
| monitoring_iostat_util     | Percentage of elapsed time during which I/O requests were issued to the device (bandwidth utilization for the device). Device saturation occurs when this value is close to 100%              | device*           | Percentage |

* device: the block device

Configuration variable `iostat_targets` can be used to specify a list of block devices to pull metrics for, space-separated.

#### meminfo (default: enabled)

Exports data from `/proc/meminfo`

| name                                   | description                                                                                                                | additional labels | units     |
|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------|-------------------|-----------|
| monitoring_meminfo_committed_as_kbytes | The number of read requests merged per second that were queued to the device                                               | none              | kibibytes |
| monitoring_meminfo_buffers_kbytes      | The amount, in kibibytes, of temporary storage for raw disk blocks                                                         | none              | kibibytes |
| monitoring_meminfo_sreclaimable_kbytes | The part of Slab that can be reclaimed, such as caches                                                                     | none              | kibibytes |
| monitoring_meminfo_slab_kbytes         | The total amount of memory, in kibibytes, used by the kernel to cache data structures for its own use                      | none              | kibibytes |
| monitoring_meminfo_swaptotal_kbytes    | The total amount of swap available, in kibibytes                                                                           | none              | kibibytes |
| monitoring_meminfo_memtotal_kbytes     | Total amount of usable RAM, in kibibytes, which is physical RAM minus a number of reserved bits and the kernel binary code | none              | kibibytes |
| monitoring_meminfo_swapfree_kbytes     | The total amount of swap free, in kibibytes                                                                                | none              | kibibytes |
| monitoring_meminfo_memfree_kbytes      | The amount of physical RAM, in kibibytes, left unused by the system                                                        | none              | kibibytes |
| monitoring_meminfo_cached_kbytes       | The amount of physical RAM, in kibibytes, used as cache memory                                                             | none              | kibibytes |

**Additionally, systems with Kernel >3 may contain the following metrics:**

| name                                   | description                                                                                 | additional labels | units     |
|----------------------------------------|---------------------------------------------------------------------------------------------|-------------------|-----------|
| monitoring_meminfo_memavailable_kbytes | An estimate of how much memory is available for starting new applications, without swapping | none              | kibibytes |

#### netdev (default: enabled)
   
Exports data from `/proc/net/dev`

| name                                  | description                                                       | additional labels | units |
|---------------------------------------|-------------------------------------------------------------------|-------------------|-------|
| monitoring_netdev_rx_bytes_total      | The total number of bytes of data received by the interface       | device`1`         | N     |
| monitoring_netdev_rx_packets_total    | The total number of packets received by the interface             | device`1`         | N     |
| monitoring_netdev_rx_errs_total       | The total number of receive errors detected by the device driver  | device`1`         | N     |
| monitoring_netdev_rx_drop_total       | The total number of packets dropped by the device driver          | device`1`         | N     |
| monitoring_netdev_rx_fifo_total       | The number of FIFO buffer errors                                  | device`1`         | N     |
| monitoring_netdev_rx_frame_total      | The number of packet framing errors                               | device`1`         | N     |
| monitoring_netdev_rx_compressed_total | The number of compressed packets received by the device driver    | device`1`         | N     |
| monitoring_netdev_rx_multicast_total  | The number of multicast frames received by the device driver      | device`1`         | N     |
| monitoring_netdev_tx_bytes_total      | The total number of bytes of data transmitted by the interface    | device`1`         | N     |
| monitoring_netdev_tx_packets_total    | The total number of packets transmittedby the interface           | device`1`         | N     |
| monitoring_netdev_tx_errs_total       | The total number of transmit errors detected by the device driver | device`1`         | N     |
| monitoring_netdev_tx_drop_total       | The total number of packets dropped by the device driver          | device`1`         | N     |
| monitoring_netdev_tx_fifo_total       | The number of FIFO buffer errors                                  | device`1`         | N     |
| monitoring_netdev_tx_frame_total      | The number of packet framing errors                               | device`1`         | N     |
| monitoring_netdev_tx_compressed_total | The number of compressed packets transmitted by the device driver | device`1`         | N     |
| monitoring_netdev_tx_multicast_total  | The number of multicast frames transmitted by the device driver   | device`1`         | N     |

`1` network device

Configuration variable `netdev_targets` can be used to specify a list of interfaces to pull metrics for, space-separated.

#### netstat (default: enabled)

| name                                  | description                                                       | additional labels | units |
|---------------------------------------|-------------------------------------------------------------------|-------------------|-------|
| monitoring_netstat_established_total  | The total number of established connections                       | none              | N     |
| monitoring_netstat_established        | The number of established connections per port                    | port`1`           | N     |

NB: Some hosts (like OpenVZ/LXC hypervisors) may have trouble reporting all connections on time with Netstat. It is advised to disable per-port granularity inside monitoring_exporter, under `netstat_totalonly`, ie. set it to `1`.

`1` source port

#### file_nr (default: enabled)

Provides data from /proc/sys/fs/file-nr

| name                         | description                     | additional labels | units |
|------------------------------|---------------------------------|-------------------|-------|
| monitoring_file_nr_allocated | All allocated file descriptors  | none              | N     |
| monitoring_file_nr_free      | Free allocated file descriptors | none              | N     |
| monitoring_file_nr_max       | Maximum open file descriptors   | none              | N     |

#### slabinfo (default: disabled)

From /proc/slabinfo - usually lots of output, so disabled by default

By default, `slab_nonzeroonly` is set to `1` so that this function reports only non-zero slabs

| name                              | description                                                                           | additional labels | units |
|-----------------------------------|---------------------------------------------------------------------------------------|-------------------|-------|
| monitoring_slabinfo_active_objs   | Number of objects that are currently active (i.e., in use)                            | slab`1`           | N     |
| monitoring_slabinfo_num_objs      | Total number of allocated objects (i.e., objects that are both in use and not in use) | slab`1`           | N     |
| monitoring_slabinfo_objsize_bytes | Size of objects in this slab, in bytes                                                | slab`1`           | Bytes |
| monitoring_slabinfo_objperslab    | Number of objects stored in each slab                                                 | slab`1`           | N     |
| monitoring_slabinfo_pagesperslab  | Number of pages allocated for each slab                                               | slab`1`           | N     |

`1` slab identifier

#### conntrack (default: disabled)

| name                              | description                | additional labels    | units |
|-----------------------------------|----------------------------|----------------------|-------|
| monitoring_conntrack_counter      | Output from `conntrack -C` | table`1`             | N     |
| monitoring_conntrack_statistics   | Output from `conntrack -S` | table`1`, counter`2`, cpu`3` | N     |

`1` conntrack table name   
`2` conntrack counter name   
`3` this label represents the CPU ID if `conntrack_sum_all_cpu` is set to `0`, otherwise the label is not present   

Configuration variable `conntrack_ignore_fake_cpuid` controls whether "fake cpuids" are reported. Default `1`.

## Extending

Instead of modifying this exporter you may be interested in creating your own workflow following this post: [Exporting Prometheus Metrics with Bash Scripts](https://apawel.me/exporting-prometheus-metrics-with-bash-scripts/)

It is relatively simple to add your own metric functions. Take on this function as an example.

```
#
# BEGIN filesystem usage
function metrics_filesystem {
    buffer=$(df -PT $filesystem_targets 2>/dev/null | grep -v '^Filesystem' | tr -s '\t' ' ')
    
    # Filesystem 1024-blocks Used Available Capacity Mounted on
    # /dev/mapper/cl-home 99533328 37320564 62212764 38% /
    buildMetric "filesystem_total_kbytes" "gauge" "From df"
    buildMetric "filesystem_used_kbytes" "gauge" "From df"
    buildMetric "filesystem_avail_kbytes" "gauge" "From df"
    buildMetric "filesystem_capacity_percent" "gauge" "From df"

    while read -r device fstype total used avail percent target; do 
        labels="{mountpoint=\"$target\",source=\"$device\",fstype=\"$fstype\"}"
        addMetricContent "filesystem_total_kbytes" "$labels $total"
        addMetricContent "filesystem_used_kbytes" "$labels $used"
        addMetricContent "filesystem_avail_kbytes" "$labels $avail"
        noPercentSign=$(echo "$percent" | sed 's,%,,g')
        addMetricContent "filesystem_capacity_percent" "$labels $noPercentSign"
    done <<< "$buffer"

    buildOutput
}
# END filesystem usage
```

The function **buildMetric** basically adds a metric to the collection, in this case the metric name is uptime_seconds_total, the type is "counter" and the help text is "System uptime".

The **addMetricContent** function is used to add the actual metric value and labels. Unfortunately there is no built-in handling for labels.

**buildOutput** is used to "convert" the metric arrays to Prometheus compatible output and is placed at the very end.

The function name name must be preceded by metrics_, ie. metrics_myfunction for it to be utilized. Once you have your function add it to the array at line 22:

`declare -a exporters=( myfunction ... )`

Note in the above array we are not using the "metrics_" prefix.
