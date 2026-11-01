# When a Corrupted WAL Fills Your Prometheus PVC

A Prometheus pod in Kubernetes suddenly starts crashlooping with `no space left on device`. You look at the volume and most of it is taken by `/prometheus/wal`. This article explains how a single corrupted WAL file can get you there, and walks through three ways out. The first gets Prometheus running as fast as possible, the second saves more data, and the third helps you understand what went wrong.

## How the WAL normally stays small

Prometheus keeps the last ~2–3 hours of samples in memory (the *head block*). Every incoming sample is also appended to the write-ahead log in `/prometheus/wal`, which is a series of numbered 128 MB segment files (`00000000`, `00000001`, ...). If Prometheus crashes, it rebuilds the head by replaying the WAL.

The WAL stays small because of a regular cleanup cycle:

1. Every ~2h the head is compacted into a persistent block (`01H...` directory).
2. Prometheus writes a **checkpoint** (`wal/checkpoint.NNNNNNNN`) summarising the older segments that are still needed.
3. The segments covered by the checkpoint are deleted.

## How corruption breaks it

To create a checkpoint, Prometheus has to **read** the old segments. If one of them is corrupted, for example by a torn write after a disk-full event or an OOM kill, checkpoint creation fails. Without a checkpoint, the old segments can't be truncated.

From that point the WAL only grows. Each new segment adds 128 MB, while nothing is deleted. Eventually the PVC fills up, writes start failing, and the pod crashloops.

Useful signals before it gets that far:

```promql
increase(prometheus_tsdb_checkpoint_creations_failed_total[1h]) > 0
increase(prometheus_tsdb_wal_truncations_failed_total[1h]) > 0
increase(prometheus_tsdb_wal_corruptions_total[1h]) > 0
```

## Why a restart is always part of the fix

Each of the solutions below ends with restarting Prometheus, and the restart is required for four reasons.

- **The WAL is only read at startup.** Replay and the built-in repair (which truncates the WAL at a bad record) only run when the TSDB is opened. A running Prometheus never reconsiders the WAL it already has on disk.
- **Deleted files don't free space while they're open.** Prometheus keeps the current segment open and `mmap`s files in `chunks_head/`. If you `rm` them under a running process, the directory entry disappears, but the disk blocks stay allocated until the file handle is closed. `df` won't move until the process exits.
- **Memory and disk must agree.** The in-memory head still references series and samples from the segments you removed. If you edit the files while Prometheus runs, the next checkpoint or compaction works from a state that no longer matches the disk, which can cause more corruption.
- **The lock file.** Prometheus holds `/prometheus/lock` exclusively. A clean stop and start avoids two writers touching the same directory.
So the safe pattern is always: **stop Prometheus → change files → start Prometheus**.

### Stopping Prometheus and getting to the files

With prometheus-operator, scale through the Prometheus resource. If you scale the StatefulSet directly, the operator reverts it.

```bash
kubectl patch prometheus k8s -n monitoring --type merge -p '{"spec":{"replicas":0}}'
```

A crashlooping pod can't be `exec`'d into reliably, so mount the PVC in a helper pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prom-wal-fix
  namespace: monitoring
spec:
  securityContext:
    runAsUser: 65534   # Prometheus runs as nobody
    fsGroup: 65534
  containers:
  - name: sh
    image: prom/prometheus   # includes a shell and promtool
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /prometheus
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: prometheus-k8s-db-prometheus-k8s-0
```

```bash
kubectl exec -it -n monitoring prom-wal-fix -- sh
```

When you're done, delete the helper pod and set `replicas` back to `1`.

## Option 1: Drop the whole WAL (get Prometheus alive)

This is the fastest and most reliable option.

```sh
rm -rf /prometheus/wal /prometheus/chunks_head
rm -f /prometheus/lock
```

On startup Prometheus creates an empty WAL and head and loads the existing blocks as usual.

- **You lose** the data that was still only in the head, roughly the last 2–3 hours. If checkpointing had been failing for a while, the loss may be longer: everything since the last successful compaction.
- **You keep** all persisted blocks, which is your history.
Use this option when monitoring has to come back now and a gap in recent data is acceptable.

## Option 2: Remove segments up to the broken one (keep the newest data)

If the corrupted segment is older and newer segments are healthy, you can keep the recent data. Prometheus usually names the bad segment in its logs:

```
corruption in segment /prometheus/wal/00001234 at 5636096: unexpected checksum
```

Remove the checkpoint and every segment up to and including the broken one:

```sh
cd /prometheus/wal
mkdir -p /prometheus/_broken
mv checkpoint.* /prometheus/_broken/
for f in [0-9]*; do [ ! "$f" \> "00001234" ] && mv -v "$f" /prometheus/_broken/; done
rm -f /prometheus/lock
```

Moving the files instead of deleting them lets you go back to Option 1, or analyse them (Option 3). If space is the problem, copy them off the volume first, then delete them.

The caveat is that WAL records are not self-contained. Series are defined once, in the segment where they first appeared, and later samples only refer to them by ID. If a series was defined in a segment you removed, its samples in the remaining segments can't be matched. Prometheus logs these as unknown series references and drops them. The result is partial data for the remaining time range, not a complete recovery.

The remaining segments must also be **contiguous**. Never leave a gap in the numbering, or replay fails.

> Prometheus's own repair does the opposite: it keeps everything *before* the corrupted record and discards everything *after* it. Which side to keep depends on whether the older or the newer data matters more to you.

## Option 3: Analyse the corrupted file

Do this afterwards, on a copy, with Prometheus already running from Option 1 or 2. The aim is to find the cause so it doesn't happen again.

**File sizes.** Segments are pre-allocated to 128 MB. A much smaller segment in the middle of the sequence points to a write that was cut off.

```sh
ls -l /prometheus/_broken/
```

**The area around the reported offset.** The WAL is written in 32 KB pages. Each record starts with a 7-byte header: 1 byte type, 2 bytes length and 4 bytes CRC32. Look at the bytes near the offset from the error:

```sh
dd if=00001234 bs=1 skip=5636096 count=256 2>/dev/null | hexdump -C
```

- **Long runs of zeros** usually mean the page was allocated but never written, because the disk filled up or the process was killed mid-write.
- **Random-looking data with a CRC mismatch** points to the storage layer: a faulty disk, or NFS and other network filesystems that Prometheus doesn't support.
**Check the persisted blocks too.** `promtool` doesn't read WAL segments, but it can confirm the blocks are healthy:

```sh
promtool tsdb list /prometheus
```

**Match the timestamps against events** from around when the segment was written:

```bash
kubectl describe pod prometheus-k8s-0 -n monitoring   # OOMKilled? Evicted?
kubectl get events -n monitoring --sort-by=.lastTimestamp
```

Typical root causes are a full disk, an OOM kill during a write, too short a `terminationGracePeriodSeconds`, unsupported storage, or two Prometheus instances writing to the same volume.

## Preventing a repeat

- Set **`retentionSize`** (`--storage.tsdb.retention.size`) to about 80% of the PVC, so blocks are removed before the disk fills.
- Alert on PVC usage (`kubelet_volume_stats_available_bytes`) and on the checkpoint and truncation failure metrics above.
- Give Prometheus enough memory and keep cardinality in check. An OOM kill during a write is a classic way to corrupt a segment.
- Use block storage with `ReadWriteOnce`, not NFS.

## Summary

| Option | Data loss | When to use |
|---|---|---|
| 1. Drop `/prometheus/wal` | Everything in the head (~2–3h or more) | Prometheus must be up now |
| 2. Remove segments up to the broken one | Older head data, plus some unmatched samples | Newer data matters and later segments are healthy |

In every case, stop Prometheus first and start it again afterwards. The WAL is only replayed at startup, and disk space is only freed once the process closes its files.
