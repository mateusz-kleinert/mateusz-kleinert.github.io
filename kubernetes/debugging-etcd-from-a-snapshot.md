# Debugging etcd from a snapshot

When etcd grows quickly, compacts slowly, or makes the kube-apiserver sluggish, the cause is usually a small number of keys. Either they are rewritten constantly, or they are very large, or both. You can find them offline from a snapshot file, without adding load to the live cluster.

Two tools help:

- **etcd-dump-db** ([etcd-io/etcd/tools/etcd-dump-db](https://github.com/etcd-io/etcd/tree/main/tools/etcd-dump-db)) reads the raw bbolt database, including every stored revision.
- **auger** ([etcd-io/auger](https://github.com/etcd-io/auger)) decodes Kubernetes objects and lists keys with their metadata.

The examples below use a snapshot called `snap.db`, for example one taken from your backups or copied from a member's `member/snap/db`. **Always work on a copy.**

## 1. Understand the layout

```bash
mkdir -p snapshot/member/snap
cp snap.db snapshot/member/snap/db
etcd-dump-db list-bucket snapshot/
# 

etcd-dump-db iterate-bucket snapshot/ meta --decode
```

All data lives in the `key` bucket, one entry per revision of each key:

```
rev={main:81234 sub:0}, value=[key "/registry/<group>/<resource>/<ns>/<name>" | val "{...}" | created 80011 | mod 81234 | ver 57]
```

## 2. Find keys with the most revisions

**Raw history (etcd-dump-db).** This counts the revisions still present in the snapshot, meaning everything since the last compaction:

```bash
etcd-dump-db iterate-bucket snapshot/ key --decode \
  | sed -n 's/.*value=\[key "\([^"]*\)" | val .*/\1/p' \
  | sort | uniq -c | sort -rn | head -20
```

Group the counts by resource type to see which kind is noisy:

```bash
etcd-dump-db iterate-bucket snapshot/ key --decode \
  | sed -n 's/.*value=\[key "\/registry\/\([^"]*\)" | val .*/\1/p' \
  | cut -d/ -f1-2 | sort | uniq -c | sort -rn | head
```

**Version counter (auger).** The kube-apiserver compacts etcd every 5 minutes by default, so raw history is often short. The `version` field counts every modification since the key was created and survives compaction, which makes it the better long-term churn signal:

```bash
auger extract -f snap.db --fields=version,valueSize,modRevision,key | sort -rn | head -20
```

## 3. Find the biggest keys

```bash
# Largest current values
auger extract -f snap.db --fields=valueSize,key | sort -rn | head -20

# Approximate bytes across all stored revisions per key
etcd-dump-db iterate-bucket snapshot/ key --decode \
  | awk -F' \\| ' '{k=$1; sub(/.*key "/,"",k); sub(/"$/,"",k); sz[k]+=length($2)} END{for(k in sz) print sz[k], k}' \
  | sort -rn | head -20
```

Many revisions multiplied by a large value is what inflates the database between compactions and defragmentations.

## 4. Inspect a suspicious key

```bash
KEY=/registry/<group>/<resource>/<namespace>/<name>

# Current value, decoded (protobuf for built-in types, JSON for CRDs)
auger extract -f snap.db -k $KEY -o json | jq .

# How it changed over time
auger extract -f snap.db -k $KEY --list-versions
auger extract -f snap.db -k $KEY --revision <rev> -o json | jq .
```

Comparing consecutive revisions usually shows what keeps changing, whether that's `status`, an annotation, `managedFields`, or the spec itself. That points to the controller responsible.

## Example: Kyverno UpdateRequests

Kyverno creates `UpdateRequest` (UR) objects to process generate and mutate-existing rules in the background. A misbehaving policy can produce thousands of URs, or a few URs that are retried endlessly. Either pattern shows up clearly in the steps above, with keys under `/registry/kyverno.io/updaterequests/`.

Count the URs and rank them by churn:

```bash
auger extract -f snap.db --keys-by-prefix /registry/kyverno.io/updaterequests/ | wc -l

auger extract -f snap.db --fields=version,valueSize,key \
  | grep updaterequests | sort -rn | head
```

### Status: why the UR isn't settling

```bash
KEY=/registry/kyverno.io/updaterequests/kyverno/ur-7x2kq
auger extract -f snap.db -k $KEY -o json | jq '{
  state:     .status.state,          # Pending | Failed | Completed | Skip
  message:   .status.message,        # error from the background controller
  retries:   .status.retryCount,
  generated: .status.generatedResources
}'
```

Watch for these patterns:

- A high `retryCount` with a repeating `message`, such as RBAC denied, "already exists", or a webhook timeout.
- A UR that stays `Pending` while its `version` keeps climbing.

### Trigger: why the UR was created

```bash
auger extract -f snap.db -k $KEY -o json | jq '{
  type:      .spec.requestType,      # generate | mutate
  policy:    .spec.policy,
  rule:      .spec.rule,
  trigger:   .spec.resource,         # kind/namespace/name that fired the rule
  operation: .spec.context.admissionRequestInfo.operation,
  user:      .spec.context.userInfo.userInfo.username,
  created:   .metadata.creationTimestamp
}'
```

- `spec.policy` and `spec.rule` identify the rule responsible.
- `spec.resource` is the object that triggered the rule.
- The admission operation and username show who touched that object. A controller service account that updates the trigger resource in a loop is a classic cause.
