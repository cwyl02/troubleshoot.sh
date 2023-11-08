---
title: Run
description: Run a specified command and output the results to a file.
---
## Run Collector

The `run` collector runs the specified command and includes the results in the collected output.

### Parameters

In addition to the [shared collector properties](/collect/collectors/#shared-properties), the `run` collector accepts the following parameters:

##### `command` (Required)
The command to execute on the host.  The command gets executed directly and is not processed by a shell.  You can specify a shell if you want to use constructs like pipes, redirection, loops, etc.  Note that if you want to run your command in a shell, then your command should be a single string argument passed to something like `sh -c`.  See the `run-with-shell` example.

##### `args` (Required)
The arguments to pass to the specified command.

##### `outputDir` (Optional)
The directory that you command to write output to on the host, if you want to include your command run's file output into your bundle. If defined, an environment variable `WORKSPACE_DIR` will be available to your command run.

##### `config` (Optional)
The path of config file that you wish to feed into your command run. It must be define as a single multi-line string. If defined, an environment variable `CONFIG` will be available to your command run. Note that this is a simple map[string]string.

## Example Collector Definition

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: run
spec:
  hostCollectors:
    - run:
        collectorName: "ping-google"
        command: "ping"
        args: ["-c", "5", "google.com"]
    - run:
        collectorName: "run-with-shell"
        command: "sh"
        args: ["-c", "du -sh | sort -rh | head -5"]
    # Multiline shell script
    - run:
        collectorName: "hostnames"
        command: "sh"
        args:
          - -c
          - |
            echo "hostname = $(hostname)"
            echo "/proc/sys/kernel/hostname = $(cat /proc/sys/kernel/hostname)"
            echo "uname -n = $(uname -n)"
    # Redirect stderr to stdout
    - run:
        collectorName: "docker-logs-etcd"
        command: "sh"
        args: ["-c", "docker logs $(docker ps -a --filter label=io.kubernetes.container.name=etcd -q -l) 2>&1"]
    # Run command with config gile and collect the file output
    - run:
        collectorName: "enriched-audit-logs"
        # must present on where you run the troubleshoot command
        command: "enrich-log.sh"
        args: ["--timeout", "10m", "--output-dir", "$WORKSPACE_OUTPUT"]
        config:
          dummy.conf: |-
            [hello]
            hello = 1

            [bye]
            bye = 2
```

## Example Collector Definition With Analyzer

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: run
spec:
  hostCollectors:
    - run:
        collectorName: "ping-google"
        command: "ping"
        args: ["-c", "5", "google.com"]
  analyzers:
    - textAnalyze:
        checkName: "run-ping"
        fileName: host-collectors/run-host/ping-google.txt
        regexGroups: '(?P<Transmitted>\d+) packets? transmitted, (?P<Received>\d+) packets? received, (?P<Loss>\d+)(\.\d+)?% packet loss'
        outcomes:
          - pass:
              when: "Loss < 5"
              message: Solid connection to google.com
          - fail:
              message: High packet loss
```

### Included Resources

The results of the `run` collector are stored in the `host-collectors/run-host` directory of the bundle. Two files per collector execution will be stored in this directory

- `[collector-name].txt` - output of the command read from `stdout`
- `[collector-name]-info.json` - the command that was executed, its exit code and any output read from `stderr`. See example below
  ```json
  {
    "command": "/sbin/ping -c 5 google.com",
    "exitCode": "0",
    "error": ""
  }
  ```

_NOTE: If the `collectorName` field is unset, it will default to `run-host`._

Example of the resulting files:

```
# ping-google.txt
PING google.com (***HIDDEN***) 56(84) bytes of data.
64 bytes from bh-in-f113.1e100.net (***HIDDEN***): icmp_seq=1 ttl=118 time=2.17 ms
64 bytes from bh-in-f113.1e100.net (***HIDDEN***): icmp_seq=2 ttl=118 time=1.29 ms
64 bytes from bh-in-f113.1e100.net (***HIDDEN***): icmp_seq=3 ttl=118 time=1.36 ms
64 bytes from bh-in-f113.1e100.net (***HIDDEN***): icmp_seq=4 ttl=118 time=1.25 ms
64 bytes from bh-in-f113.1e100.net (***HIDDEN***): icmp_seq=5 ttl=118 time=1.31 ms

--- google.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 1.252/1.478/2.171/0.348 ms
```

and

```
# run-with-shell.txt
3.4G    /var/lib/kurl/assets
1.3G    /var/lib/kurl/assets/rook-1.5.9.tar.gz
1.1G    /var/log/apiserver
897M    /var/lib/kurl/assets/kubernetes-1.19.16.tar.gz
812M    /var/lib/kurl/assets/docker-20.10.5.tar.gz
```
