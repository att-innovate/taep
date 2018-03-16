## TAEP - Traffic Analysis and Experimentation
This document provides a few sample scenarios for our Traffic Analysis and Experimentation Platform.

### Lab Setup
All the scenarios in this document are based on following setup.
  
#### Cabling
The EdgeCore-Barefoot switch is cabled as follow:

![cabling](https://github.com/att-innovate/taep/blob/master/assets/cabling.jpg?raw=true)

Port 0 and Port 4 are connected to the production network. Port 8 is connected straight to port 12, and port 16 is connected to port 20.

### Config
The TAEP-Controller uses following configuration. The configuration is stored at its default location `./config/config.yml`.

	bf-bin-path: /root/bf-sde/install
	bf-config-file: /root/taep-controller/p4/l2_switching.conf
	enable-labeling: true
	api-port: 8100
	ports:
	    - number: 0
	      speed: 40
	      autoneg-disabled: false
	    - number: 4
	      speed: 40
	      autoneg-disabled: false
	    - number: 8
	      speed: 40
	    - number: 12
	      speed: 40
	    - number: 16
	      speed: 40
	    - number: 20
	      speed: 40
	connections:
	    - from: 0
	      to: 4
	      type: bidirectional
	    - from: 12
	      to: 4
	      type: unidirectional
	    - from: 20
	      to: 4
	      type: unidirectional
	    - from: 8
	      to: 0
	      type: unidirectional
	    - from: 16
	      to: 0
	      type: unidirectional
	hhd:
	    analysis-window-in-seconds: 30
	    max-number-of-flows: 200

All the ports are configured as 40G ports. The ports connected to the production network have “auto-negotiation” turned on.

The configured connections form following network setup.

![connections](https://github.com/att-innovate/taep/blob/master/assets/connections.jpg?raw=true)

The bi-directional 0-4 connection guarantees that all production traffic will by default flow unchanged straight through the switch.

All the uni-directional connection definitions guarantee that packets diverted to one of the parallel paths will get forwarded back in to the production network. Example: A packet diverted from 0 to 8 will egress port 8 and arrive back in to the switch at 12, from where it is forwarded to 4, back in to the production network.

### Forward Production Traffic
With the configuration in place we can connect port 0 and 4 to the production network and start the TAEP-Controller.

As implemented in our [P4 code](https://github.com/att-innovate/taep-controller/blob/master/p4/l2_switching/l2_switching.p4) packets simply get passed through without any modification. 

#### Start TAEP
Start TAEP Controller:

	$ service taep start

Check the log-file:

	$ cd /root/taep-scripts
	$ ./scripts/log-controller.sh

Output you should get:

	Port: 168 added with fec disabled true, with status: 0
	autoneg_policy enabled, status 0
	port enabled status 0
	Added entry to Forwarding Table, Handle 1
	Added entry to Forwarding Table, Handle 2
	Added entry to Forwarding Table, Handle 3
	Added entry to Forwarding Table, Handle 4
	Added entry to Forwarding Table, Handle 5
	Added entry to Forwarding Table, Handle 6
	HHD Max Number of Flow set to 200
	HHD Analysis Window 30s
	Callback function for flow learning registered

The log file is at `/var/log/taep`.

Btw, port numbers shown in the log are the internal port numbers converted from our configured port numbers.

Start TAEP-Analytics

	$ cd /root/taep-scripts
	$ ./scripts/run-analytics.sh

Output lists all the containers started by Docker-Compose.

	Creating docker_kapacitor_1
	Creating docker_telegraf_1
	Creating docker_agent_1
	Creating docker_grafana_1
	Creating docker_influxdb_1

#### Connect to Live Network

Verify that port speed on surrounding network devices are configured accordingly (40G, auto-neg on) and connect the cable.

#### Verify setup using Grafana-Dashboards

The individual Grafana Dashboards share the same IP address as your terminal session to the switch.

Example URL for Grafana: http://ip-barefoot:8082

By default username/password is set to admin/admin

Dashboards can be accessed via the “Home” pulldown menu top left in the header of Grafana.

**Network Dashboard**

The Network Dashboard shows the production network traffic passing through, in either “packets/s” or “mbytes/s”.

![network-dashboard](https://github.com/att-innovate/taep/blob/master/assets/network-dashboard.jpg?raw=true)

**Docker Dashboard**

The Docker Dashboard shows memory and cpu usage of each individual container of our Analytics Stack.

![docker-dashboard](https://github.com/att-innovate/taep/blob/master/assets/docker-dashboard.jpg?raw=true)

**System Health Dashboard**

The System Health Dashboard shows memory, cpu, and disk usage for the micro-server itself. The “Health-Index”, 10 being healthiest, gets re-calculated every 5 minutes.

![system-health-dashboard](https://github.com/att-innovate/taep/blob/master/assets/system-health-dashboard.jpg?raw=true)

### Simple Network Traffic Analysis

Packets can be diverted based on individual IP addresses or IP ranges. This enables us to collect on demand network metrics for individual flows. The metrics get stored in our timeseries database with corresponding labels for further analysis.

Example: Divert traffic with destination 10.250.3.0/24 incoming at port 0 through port 16-20.

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 0, "port_egress": 16, "ip_address": "10.250.3.0", "ip_prefix_length": 24}' 'http://localhost:8100/divert/dest'

Using the “Network - TAEP” Dashboard you should be able to see the diverted traffic in the “Data 16 -> 20” chart.

The “Divert Table 0 -> 16” shows the timestamped “label” that got inserted in to the timeseries database.

Btw, in our test setup most of the traffic is going southbound, 0 to 4, as visualized in the “Data In 0” chart, very little traffic is going the other direction, “Data In 4”.

![network-divert-dashboard](https://github.com/att-innovate/taep/blob/master/assets/network-divert-dashboard.jpg?raw=true)

All the divert rules can be deleted by calling:

	$ curl -X DELETE 'http://barefoot-32-mgmt:8100/divert'	

### Flow Learning
TAEP comes with a simple “Flow Learner” function. During a defined time-window TAEP will collect all the flows arriving at a specific port.

A Flow is defined as a unique combination of (Src IP, Dest IP, Protocol Type (UDP/TCP), Src Port, Dest Port).

Example: Collect up to 500 unique flows arriving at port 0 during the next 10 seconds.

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 0, "max_number_of_flows": 500, "time_window_in_seconds": 10}' 'http://localhost:8100/flows'

A list of learned flows can afterwards be retrieved by calling:

	$ curl http://localhost:8100/flows

Example response:

	[{"src_addr":"10.250.3.25","src_addr_int":184156953,"src_port":39863,"dst_addr":"10.250.3.22","dst_addr_int":184156950,"dst_port":22,"ipv4_protocol":6,"hash1":1019,"hash2":13420},{"src_addr":"10.250.3.25","src_addr_int":184156953,"src_port":48118,"dst_addr":"10.250.3.24","dst_addr_int":184156952,"dst_port":22,"ipv4_protocol":6,"hash1":8897,"hash2":3094} ...]

The list of flows is cached until a next flow learning session gets started.

## Experiment: Divert Heavy Hitter
We implemented a simple divert function that automatically diverts “heavy flows”.

A “heavy flow” is defined as the flow with the highest packet count during a pre-defined time-window.

Example: We use two divert paths 8->12 and 16->20. We divert a subset of the traffic (10.250.3.0/24) arriving at port 0 through 8->12. We use an analysis window of 30s, as defined in `config/config.yml`. During that window our Heavy Hitter Detector looks for the heaviest flow in both the 8->12 and the 16->20 path. For the following time-window, for the next 30s, that flow gets diverted through 16->20 while the system looks for the next heavy flow.

Divert traffic for 10.250.3.0/24 through 8->12

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 0, "port_egress": 8, "ip_address": "10.250.3.0", "ip_prefix_length": 24}' 'http://localhost:8100/divert/dest'

Start the Heavy Hitter Diverting function on the diverted traffic:

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 12, "port_ingress_divert": 20, "divert_ingress": 0, "divert_egress": 16}' 'http://localhost:8100/hhd/dest'

At some time during the experiment we manually added additional heavy traffic generated by iperf.

![network-hhd-dashboard](https://github.com/att-innovate/taep/blob/master/assets/network-hhd-dashboard.jpg?raw=true)

“Data In 0” shows all the traffic arriving at port 0. “Data 8-12” shows the traffic that gets diverted from 0 to 8. “Data 16-20” displays the traffic that automatically gets diverted by the HHD function. “Divert Table 0->16” lists the IP addresses of the heavy flows getting diverted during each time window.

For the demo the default heavy hitter was the flow with destination 10.250.3.24. During the experiment we run an additional heavy flow with destination 10.250.3.21. This heavy flow, as can be seen in the charts, first gets diverted through the standard path until it gets picked up by the HHD function and automatically diverted through 16-20.

The HHD function and all divert rules can be deleted by calling:

	$ curl -X DELETE 'http://barefoot-32-mgmt:8100/divert'	
	$ curl -X DELETE 'http://barefoot-32-mgmt:8100/hhd'

## Experiment: AI/ML Agent
In our production setup we implemented and successfully tested a Reinforcement Learning Agent. The Agents was used to balance traffic between 8-12 and 16-20 in a pre-defined ratio.

Currently we are not ready to open source this RL code, but the project comes with the necessary scaffolding code for such an Agent.

A detailed description can be found in the related [TAEP-Analytics](https://github.com/att-innovate/taep-analytics) sub-project. This project also contains the [Agent code](https://github.com/att-innovate/taep-analytics/tree/master/docker/agent) itself.

The code is written in Python and it can easily be extended with existing AI/ML libraries.

Turn on metrics forwarding to Agent:

    $ cd /root/taep-analytics
    $ ./scripts/enable-kapacitor-agent.sh

Check Agent Log for the metrics being pushed

    $ cd /root/taep-analytics
    $ ./scripts/log-agent.sh

Output

    2018-03-01 23:25:11,714 INFO:root: Begin Batch
    Point.tags : port_0
    Point.octets_in : 158539452.0
    Point.octets_out : 862102.0
    Point.packets_in : 110261.0
    Point.packets_out : 11645.0
    Point.time : 1519946690000000000
    Point.tags : port_0
    Point.octets_in : 2060.0
    Point.octets_out : 3353.0
    Point.packets_in : 32.0
    Point.packets_out : 52.0
    ....

Those metrics serve as the input to a typical control loop, to any "smart" balancing algorithm. The algorithm can use the [API](https://github.com/att-innovate/taep-controller#rest-api) of the controller to implement correcting actions, and subsequently verify its impact by observing the stream off metrics.

![taep-stack](https://github.com/att-innovate/taep/blob/master/assets/taep-stack.jpg?raw=true)

Turn of the metrics forwarding to Agent:

    $ cd /root/taep-analytics
    $ ./scripts/enable-kapacitor-agent.sh

## Summary
No packets got hurt during all of our experiments. The platform allows us to do those experiments in a controlled self-contained way without interfering with the production traffic itself. We believe that this provides a unique and attractive way to better understand flow behavior in any production network.