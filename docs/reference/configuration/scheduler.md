---
id: scheduler
title: Scheduler
---

## Configure Scheduler YAML File {#configure-scheduler-yaml-file}

The default path for the scheduler yaml configuration file is `/etc/dragonfly/scheduler.yaml` in linux,
and the default path is `$HOME/.dragonfly/config/scheduler.yaml` in darwin.

```yaml
# server scheduler instance configuration
server:
  # # ip
  # ip: 127.0.0.1
  # # host
  # host: localhost
  # port is the ip and port scheduler server listens on.
  port: 8002
  # cacheDir is dynconfig cache storage directory
  # in linux, default value is /var/cache/dragonfly
  # in macos(just for testing), default value is /Users/$USER/.dragonfly/cache
  cacheDir: ''
  # logDir is the log storage directory
  # in linux, default value is /var/log/dragonfly
  # in macos(just for testing), default value is /Users/$USER/.dragonfly/logs
  logDir: ''

# scheduler policy configuration
scheduler:
  # algorithm configuration to use different scheduling algorithms,
  # default configuration supports "default" and "ml"
  # "default" is the rule-based scheduling algorithm,
  # "ml" is the machine learning scheduling algorithm
  # It also supports user plugin extension, the algorithm value is "plugin",
  # and the compiled `d7y-scheduler-plugin-evaluator.so` file is added to
  # the dragonfly working directory plugins
  algorithm: default
  # backSourceCount is the number of backsource clients
  # when the CDN is unavailable
  backSourceCount: 3
  # retry scheduling back-to-source limit times
  retryBackSourceLimit: 5
  # retry scheduling limit times
  retryLimit: 10
  # retry scheduling interval
  retryInterval: 50ms
  # gc metadata configuration
  gc:
    # peerGCInterval is peer's gc interval
    peerGCInterval: 10m
    # peerTTL is peer's TTL duration
    peerTTL: 24h
    # taskGCInterval is task's gc interval
    taskGCInterval: 10m
    # taskTTL is task's TTL duration
    taskTTL: 24h
    # hostGCInterval is host's gc interval
    hostGCInterval: 30m
    # hostTTL is host's TTL duration
    hostTTL: 48h

# dynamic data configuration
dynConfig:
  # dynamic config refresh interval
  refreshInterval: 1m

# scheduler host configuration
host:
  # idc is the idc of scheduler instance
  idc: ''
  # netTopology is the net topology of scheduler instance
  netTopology: ''
  # location is the location of scheduler instance
  location: ''

# manager configuration
manager:
  # addr manager access address
  addr: 127.0.0.1:65003
  # schedulerClusterID cluster id to which scheduler instance belongs
  schedulerClusterID: 1
  # keepAlive keep alive configuration
  keepAlive:
    # interval
    interval: 5s

# cdn configuration
cdn:
  # scheduler enable cdn as P2P peer,
  # if the value is false, P2P network will not be back-to-source through
  # cdn but by dfdaemon and preheat feature does not work
  enable: true

# machinery async job configuration,
# see https://github.com/RichardKnop/machinery
job:
  # scheduler enable job service
  enable: true
  # number of workers in global queue
  globalWorkerNum: 1
  # number of workers in scheduler queue
  schedulerWorkerNum: 1
  # number of workers in local queue
  localWorkerNum: 5
  # redis configuration
  redis:
    # host
    host: ''
    # port
    port: 6379
    # password
    password: ''
    # brokerDB
    brokerDB: 1
    # backendDB
    backendDB: 2

# enable prometheus metrics
metrics:
  # scheduler enable metrics service
  enable: false
  # metrics service address
  addr: ':8000'
  # enable peer host metrics
  enablePeerHost: false

# console shows log on console
console: false

# whether to enable debug level logger and enable pprof
verbose: false

# listen port for pprof, only valid when the verbose option is true
# default is -1. If it is 0, pprof will use a random port.
pprof-port: -1

# jaeger endpoint url, like: http://jaeger.dragonfly.svc:14268/api/traces
jaeger: ''
```
