# About this repository

jtimon can collect openconfig telemetry from Junos devices  
it can also exports telemetry data received from Junos devices to Prometheus. The default export port is 8090, it is configurable.  

We will use jtimon to collect openconfig telemetry from junos devices.  
The data collected by jtimon will be exported to Prometheus

# Junos device details 

## version
```
lab@jedi-vmx-1-vcp> show version | match telemetry
```
```
lab@jedi-vmx-1-vcp> show version | match openconfig
```
## configuration 
```
lab@jedi-vmx-1-vcp> show configuration system services extension-service | display set
```
```
lab@jedi-vmx-1-vcp> show configuration system services netconf | display set
```
## troubleshooting 
to Display information about sensors, run this command: 
```
lab@jedi-vmx-1-vcp> show agent sensors
```

#  Prometheus

## About Prometheus 
https://prometheus.io/  

## Install  Prometheus
Download Prometheus
```
$ wget https://github.com/prometheus/prometheus/releases/download/v2.5.0/prometheus-2.5.0.linux-amd64.tar.gz .
```
extract the content of the archieve
```
$ tar xvfz prometheus-2.5.0.linux-amd64.tar.gz
```
```
$ cd prometheus-2.5.0.linux-amd64/
```
```
$ ./prometheus --version
$ ./prometheus --help
```
## configure Prometheus  
update the ```prometheus.yml``` configuration file  
[use this file](prometheus.yml) 
```
$ vi prometheus.yml
```
```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
 
  - job_name: 'jtimon'
    static_configs:
    - targets: ['localhost:8090']
```
## Start Prometheus
```
./prometheus --config.file=prometheus.yml
```


# jtimon

## About jtimon

https://github.com/nileshsimaria/jtimon  
https://forums.juniper.net/t5/Automation/OpenConfig-and-gRPC-Junos-Telemetry-Interface/ta-p/316090  
https://github.com/nileshsimaria/jtimon/wiki/JTIMON-and-Prometheus

## Install jtimon (docker container) 

### requirements
Install Docker

### Build a jtimon Docker image 

```
# git clone https://github.com/nileshsimaria/jtimon.git
# cd jtimon/
# make docker
```
### check the image
```
# docker images jtimon
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
jtimon              latest              2e8967d4ea00        2 hours ago         16.4 MB
```
### List running containers 
There is no container running
```
# docker ps | grep jtimon
```
## Run jtimon 
```
# ./jtimon --help
Usage of /usr/local/bin/jtimon:
      --alias-file string          File containing aliasing information
      --api                        Receive HTTP commands when running
      --compression string         Enable HTTP/2 compression (gzip, deflate)
      --config strings             Config file name(s)
      --config-file-list string    List of Config files
      --explore-config             Explore full config of JTIMON and exit
      --grpc-headers               Add grpc headers in DB
      --gtrace                     Collect GRPC traces
      --json                       Convert telemetry packet into JSON
      --latency-profile            Profile latencies. Place them in TSDB
      --log-mux-stdout             All logs to stdout
      --max-run int                Max run time in seconds
      --no-per-packet-goroutines   Spawn per packet go routines
      --pprof                      Profile JTIMON
      --pprof-port int32           Profile port (default 6060)
      --prefix-check               Report missing __prefix__ in telemetry packet
      --print                      Print Telemetry data
      --prometheus                 Stats for prometheus monitoring system
      --prometheus-port int32      Prometheus port (default 8090)
      --stats-handler              Use GRPC statshandler
      --version                    Print version and build-time of the binary and exit

```
Alternatively, run this command. These 2 commands are equivalents. 
```
# docker run -it --rm jtimon --help
```

## create a jtimon configuration file
[use this file](vmx1.json)
```
# vi vmx1.json
```
root@dc-automation:~/jtimon# more vmx1.json
{
    "host": "172.30.52.152",
    "port": 50051,
    "user": "lab",
    "password": "m0naco",
    "cid": "my-client-id",
    "grpc" : {
        "ws" : 524288
    },
    "paths": [{
        "path": "/interfaces",
        "freq": 2000
    }]
}



## Pass the jtimon configuration file to the container

lets run jtimon dockerized while passing the local directory to the container to access the configuration file.  

run jtimon with the configuration file ```vmx1.json``` and Print Telemetry data
 
```
$ ./jtimon --prometheus --prometheus-port 8090 --config vmx1.json --print --alias-file alias.txt

```
Alternatively, run this command. These 2 commands are equivalents. 

```
# docker run -it --rm -v $PWD:/u jtimon --config vmx1.json --print --prometheus --prometheus-port 8090 --alias-file alias.txt
```


alias-file is optional.  
you can make use of it to give small aliases
You can provide the mapping as shown below.

$ cat alias.txt
ifd : /interfaces/interface/name
physical_interface : /interfaces/interface/@name
ifd-admin-status : /interfaces/interface/state/admin-status
ifd-oper-status : /interfaces/interface/state/oper-status
ifd-in-pkts:/interfaces/interface/state/counters/in-pkts
ifd-in-octets:/interfaces/interface/state/counters/in-octets
ifl:/interfaces/interface/subinterfaces/subinterface/index
logical-interface-index:/interfaces/interface/subinterfaces/subinterface/@index
ifl-in-ucast-pkts:/interfaces/interface/subinterfaces/subinterface/state/counters/in-unicast-pkts
ifl-in-mcast-pkts:/interfaces/interface/subinterfaces/subinterface/state/counters/in-multicast-pkts

If JTIMON does not find alias, it would use the names of the path as received from JTI (it will replace '/' with '_').

# verify on prometheus

open a browser to prometheus http://<prometheus_ip>:9090




root@dc-automation:~/jtimon# curl -g 'http://172.30.52.37:9090/api/v1/series?match[]=up'

curl -g 'http://172.30.52.37:9090/api/v1/series?match[]=jtimon'

 curl -g 'http://172.30.52.37:9090/api/v1/label/job/values'



/interfaces/interface[name='lo0']/subinterfaces/subinterface[index='0']/state/counters/in-octets


