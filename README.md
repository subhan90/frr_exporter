# Free Range Routing (FRR) Exporter

Prometheus exporter for FRR version 3.0+ that collects metrics by using `vtysh` and exposes them via HTTP, ready for collecting by Prometheus.

## Getting Started
To run frr_exporter:
```
./frr_exporter [flags]
```

To view metrics on the default port (9342) and path (/metrics):
```
http://device:9342/metrics
```

To view available flags:
```
usage: frr_exporter [<flags>]

Flags:
  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --collector.bgp.peer-types
                                 Enable the frr_bgp_peer_types_up metric (default: disabled).
      --collector.bgp.peer-types.keys=type ...
                                 Select the keys from the JSON formatted BGP peer description of which the values will be used with the frr_bgp_peer_types_up metric. Supports multiple values (default:
                                 type).
      --collector.bgp.peer-descriptions
                                 Add the value of the desc key from the JSON formatted BGP peer description as a label to peer metrics. (default: disabled).
      --collector.bgp.peer-descriptions.plain-text
                                 Use the full text field of the BGP peer description instead of the value of the JSON formatted desc key (default: disabled).
      --collector.bgp.advertised-prefixes
                                 Enables the frr_exporter_bgp_prefixes_advertised_count_total metric which exports the number of advertised prefixes to a BGP peer. This is an option for older versions of
                                 FRR that don't have PfxSent field (default: disabled).
      --web.listen-address=":9342"
                                 Address on which to expose metrics and web interface.
      --web.telemetry-path="/metrics"
                                 Path under which to expose metrics.
      --frr.vtysh.path="/usr/bin/vtysh"
                                 Path of vtysh.
      --frr.vtysh.options=""     Additional options passed to vtysh.
      --frr.vtysh.timeout="20s"  The timeout when running vtysh commands (default: 20s).
      --frr.vtysh.sudo           Enable sudo when executing vtysh commands.
      --collector.bgp            Collect BGP Metrics (default: enabled).
      --collector.ospf           Collect OSPF Metrics (default: enabled).
      --collector.bgp6           Collect BGP IPv6 Metrics (default: disabled).
      --collector.bgpl2vpn       Collect BGP L2VPN Metrics (default: disabled).
      --collector.bfd            Collect BFD Metrics (default: enabled).
      --collector.vrrp           Collect VRRP Metrics (default: disabled).
      --log.level="info"         Only log messages with the given severity or above. Valid levels: [debug, info, warn, error, fatal]
      --log.format="logger:stderr"
                                 Set the log target and format. Example: "logger:syslog?appname=bob&local=7" or "logger:stdout?json=true"
      --version                  Show application version.
```

Promethues configuraiton:
```
scrape_configs:
  - job_name: frr
    static_configs:
      - targets:
        - device1:9342
        - device2:9342
    relabel_configs:
      - source_labels: [__address__]
        regex: "(.*):\d+"
        target: instance
```

## Docker
A Docker container is available at [tynany/frr_exporter](https://hub.docker.com/r/tynany/frr_exporter).

### Example
Mount the FRR config directory (default `/etc/frr`) and FRR socket directory (default `/var/run/frr`) inside the container, passing those directories to vtysh options `--vty_socket` & `--config_dir` via the frr_exporter option `--frr.vtysh.options`:
```
docker run --restart unless-stopped -d -p 9342:9342 -v /etc/frr:/frr_config -v /var/run/frr:/frr_sockets tynany/frr_exporter "--frr.vtysh.options=--vty_socket=/frr_sockets --config_dir=/frr_config"
```


## Collectors
To disable a default collector, use the `--no-collector.$name` flag, or
`--collector.$name` to enable it.

### Enabled by Default
Name | Description
--- | ---
BGP | Per VRF and address family (currently support unicast only) BGP metrics:<br> - RIB entries<br> - RIB memory usage<br> - Configured peer count<br> - Peer memory usage<br> - Configure peer group count<br> - Peer group memory usage<br> - Peer messages in<br> - Peer messages out<br> - Peer received prefixes<br> - Peer advertised prefixes<br> - Peer state (established/down)<br> - Peer uptime
OSPFv4 | Per VRF OSPF metrics:<br> - Neighbors<br> - Neighbor adjacencies
BFD | BFD Peer metrics:<br> - Count of total number of peers<br> - BFD Peer State (up/down)<br> - BFD Peer Uptime in seconds

### Disabled by Default
Name | Description
--- | ---
BGP IPv6 | Per VRF and address family (currently support unicast only) BGP IPv6 metrics:<br> - RIB entries<br> - RIB memory usage<br> - Configured peer count<br> - Peer memory usage<br> - Configure peer group count<br> - Peer group memory usage<br> - Peer messages in<br> - Peer messages out<br> - Peer active prfixes<br> - Peer state (established/down)<br> - Peer uptime
BGP L2VPN | Per VRF and address family (currently support EVPN only) BGP L2VPN EVPN metrics:<br> - RIB entries<br> - RIB memory usage<br> - Configured peer count<br> - Peer memory usage<br> - Configure peer group count<br> - Peer group memory usage<br> - Peer messages in<br> - Peer messages out<br> - Peer active prfixes<br> - Peer state (established/down)<br> - Peer uptime 
VRRP | Per VRRP Interface, VrID and Protocol:<br> - Rx and TX statistics<br> - VRRP Status<br> - VRRP State Transitions<br>

### VTYSH
The vtysh command is heavily utilised to extract metrics from FRR. The default timeout is 20s but can be modified via the `--frr.vtysh.timeout` flag.

### BGP: Peer Description Labels
The description of a BGP peer can be added as a label to all peer metrics by passing the `--collector.bgp.peer-descriptions` flag. The peer description must be JSON formatted with a `desc` field. Example configuration:

```
router bgp 64512
 neighbor 192.168.0.1 remote-as 64513
 neighbor 192.168.0.1 description {"desc":"important peer"}
```

If an unstructured description is preferred, additionally to `--collector.bgp.peer-descriptions` pass the `--collector.bgp.peer-descriptions.plain-text` flag. Example configuration:

```
router bgp 64512
 neighbor 192.168.0.1 remote-as 64513
 neighbor 192.168.0.1 description important peer
```

Note, it is recommended to leave this feature disabled as peer descriptions can easily change, resulting in a new time series.

### BGP: Advertised Prefixes to a Peer
This is an option for older versions of FRR. If your FRR shows the "PfxSnt" field for Peers in the Established state in the output of "show bgp summary json", you don't need to enable this option.

The number of prefixes advertised to a BGP peer can be enabled (i.e. the `frr_exporter_bgp_prefixes_advertised_count_total` metric) by passing the `--collector.bgp.advertised-prefixes` flag. Please note, FRR does not expose a summary of prefixes advertised to BGP peers, so each peer needs to be queried individually. For example, if 20 BGP peers are configured, 20 `vtysh -c 'sh ip bgp neigh X.X.X.X advertised-routes json'` commands are executed. This can be slow -- the commands are executed in parallel by frr_exporter, but vtysh/FRR seems to execute them in serial. Due to the potential negative performance implications of running `vtysh` for every BGP peer, this metric is disabled by default.

### BGP: frr_bgp_peer_types_up
FRR Exporter exposes a special metric, `frr_bgp_peer_types_up`, that can be used in scenarios where you want to create Prometheus queries that report on the number of types of BGP peers that are currently established, such as for Alertmanager. To implement this metric, a JSON formatted description must be configured on your BGP group. FRR Exporter will then use the value from the keys specific by the `--collector.bgp.peer-types.keys` flag (the default is `type`), and aggregate all BGP peers that are currently established and configured with that type.

For example, if you want to know how many BGP peers are currently established that provide internet, you'd set the description of all BGP groups that provide internet to `{"type":"internet"}` and query Prometheus with `frr_bgp_peer_types_up{type="internet"})`. Going further, if you want to create an alert when the number of established BGP peers that provide internet is 1 or less, you'd use `sum(frr_bgp_peer_types_up{type="internet"}) <= 1`.

To enable `frr_bgp_peer_types_up`, use the `--collector.bgp.peer-types` flag.

## Development
### Building
```
go get github.com/tynany/frr_exporter
cd ${GOPATH}/src/github.com/prometheus/frr_exporter
go build
```

## TODO
 - Collector and main tests
 - OSPF6
 - ISIS
 - Additional BGP SAFI
 - Feel free to submit a new feature request
