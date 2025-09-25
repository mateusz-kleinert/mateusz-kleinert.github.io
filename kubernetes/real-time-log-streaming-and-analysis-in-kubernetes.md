# Real-time log streaming and analysis in Kubernetes

## Introduction

Kubernetes, a powerful container orchestration platform, manages the lifecycle of containers. As applications become increasingly complex and distributed, effective logging is crucial for monitoring, troubleshooting, and security.

In this post I would like to focus on tools used for real-time log streaming. Real-time log analysis can help identify performance bottlenecks, errors, and anomalies. By examining logs in real-time, developers can quickly spot the root cause of problems.

## Command Line Interface

### kubectl logs

When it comes to interacting with Kubernetes cluster `kubectl` is the first tool that comes to mind. It not only allows us to manage the resources, but comes in handy for monitoring and debugging purposes as well. Let's look at some examples:

#### Examples

When there is only a single container running on a Pod we can omit the container's name and provide only the name of the Pod.

```
kubectl logs <pod>
```

In case there are multiple containers running on a single pod we can specify the container's name with `-c` option.

```
kubectl logs -c <container> <pod>
```

To get logs from multiple contianers spread accross multiple pods we can specify deployment:

```
kubectl logs deployment/<deployment> -c <container>
```

Some more options and flags:
* `--all-containers=true` - get logs from all containers running on a pod,
* `-l <label>` - allow filtering containers by label,
* `-p` - return logs from previous terminated container,
* `-f` - start streaming the logs,
* `--tail=<number>` - restrict the logs to the most recent `number` of lines,
* `--since=<time>` - restrict the logs to the most recent `time`, e.g. `--since=1h` will return logs from the last hour,
* `--timestamps=false` - include timestamps on each line of the output log,
* `--pod-running-timeout=<time>` - wait for at least one running pod for a specified amount of `time`,
* `--prefix=true` - prefix each log line with its source.

### stern

[stern](https://github.com/stern/stern) is a powerful tool that allows to tail logs from multiple pods and multiple containers within the pods. It has some useful features like:
* automatically add/remove pods from the tail (e.g. if the pod gets deleted),
* output can be formatted to e.g. `json` format with the `--output` flag,
* `~/.config/stern/config.yaml` file can be used to tweak the default values,
* with the `--template` option user can change the output to a custom template.

#### Examples

Get logs from pods selected by the `pod-query` in a given `context`. In addition, let's exclude all log lines matched with the `exclude` regular expression:

```
stern --context=<context> <pod-query> --exclude=<exclude>
```

## Choosing the right tool for the job

`kubectl logs` is the best tool for a simple log inspection. The main advantage of `kubectl` is its multifunctionality and availability. At the same time `stern` is a powerful and flexible tool for advanced log analysis and troubleshooting.
