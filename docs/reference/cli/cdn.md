---
id: cdn
title: CDN
---

CDN is a long-running process which caches downloaded data
from source to avoid downloading the same files from source repeatedly.

## Usage {#usage}

```shell
cdn [flags]
cdn [command]
```

## Available Commands {#available-commands}

```text
completion  generate the autocompletion script for the specified shell
doc         generate documents
help        Help about any command
plugin      show plugin
version     show version
```

## Flags {#flags}

<!-- markdownlint-disable -->

```text
    --config string         the path of configuration file with yaml extension name, default is /etc/dragonfly/cdn.yaml, it can also be set by env var: CDN_CONFIG
    --console               whether logger output records to the stdout
-h, --help                  help for cdn
    --jaeger string         jaeger endpoint url, like: http://localhost:14250/api/traces
    --pprof-port int        listen port for pprof, 0 represents random port (default -1)
    --service-name string   name of the service for tracer (default "dragonfly-cdn")
    --verbose               whether logger use debug level
```

<!-- markdownlint-restore -->
