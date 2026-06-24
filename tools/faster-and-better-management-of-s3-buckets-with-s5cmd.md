# Faster and Better Management of S3 Buckets with s5cmd

If you work with S3 regularly, you've probably felt the friction of `aws-cli`: slow uploads, sluggish bulk deletes, and painful iteration over thousands of objects. [`s5cmd`](https://github.com/peak/s5cmd) is a parallel S3 and local filesystem execution tool written in Go that solves exactly this problem. It's built around concurrent worker pools, so it parallelizes aggressively and pushes throughput far beyond the traditional tools — in published benchmarks it uploads roughly 32x faster than `s3cmd` and 12x faster than `aws-cli`, and can saturate a 40 Gbps link on downloads ([source: Joshua Robinson's benchmark on Medium](https://medium.com/@joshua_robinson/s5cmd-for-high-performance-object-storage-7071352cc09d)).

Beyond raw speed, `s5cmd` supports the full range of object-management tasks you'd expect — listing, copying, moving, deleting, syncing — plus first-class wildcard support, batched delete requests, a "run" mode for executing thousands of commands at once, and compatibility with any S3-API service such as Google Cloud Storage or MinIO.

## Getting Started

Installation is straightforward. On macOS:

```bash
brew install peak/tap/s5cmd
```

Pre-built binaries for Linux, macOS, and Windows are available on the project's releases page, and it's also packaged for Docker, Conda, and several other platforms.

## Examples

### 1. Configuration / credentials

`s5cmd` does **not** have its own configuration file. Instead, it uses the official AWS SDK and reads the standard AWS credential and config files — the same ones `aws-cli` uses. To set it up, populate `~/.aws/credentials`:

```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY

[my-work-profile]
aws_access_key_id = ANOTHER_ACCESS_KEY_ID
aws_secret_access_key = ANOTHER_SECRET_ACCESS_KEY
```

You can then select a profile per command, or rely on environment variables instead:

```bash
# Use a named profile
s5cmd --profile my-work-profile ls s3://my-company-bucket/

# Or via environment variables
export AWS_ACCESS_KEY_ID='<your-access-key-id>'
export AWS_SECRET_ACCESS_KEY='<your-secret-access-key>'
export AWS_REGION='<your-bucket-region>'
s5cmd ls s3://your-bucket/
```

### 2. List buckets and objects

Run `ls` with no argument to list all your buckets, or point it at a bucket/prefix to list objects:

```bash
# List all buckets
s5cmd ls

# List objects under a prefix
s5cmd ls s3://my-bucket/logs/2020/03/
```

### 3. Batch remove files with a wildcard

This is where `s5cmd` shines. A single wildcard `rm` matches all objects under a prefix and removes them using S3's batch delete API (up to 1,000 objects per request):

```bash
s5cmd rm 's3://my-bucket/logs/2020/03/19/*'
```

> Wrap wildcard expressions in single quotes so your shell (e.g. zsh) doesn't try to expand `*` locally.

### 4. Copy with wildcards

Download or upload many objects in parallel. The source directory structure is preserved by default, or you can flatten it:

```bash
# Download everything under a prefix into ./logs/
s5cmd cp 's3://my-bucket/logs/2020/03/*' logs/

# Upload a whole local directory
s5cmd cp directory/ s3://my-bucket/
```

### 5. Run thousands of commands in batch

The most powerful feature: declare many operations in a file (or pipe them in) and `s5cmd` executes them across parallel workers from a single process, reaching thousands of operations per second.

```bash
s5cmd run commands.txt
```

Where `commands.txt` might contain:

```text
cp 's3://my-bucket/2020/03/*' logs/2020/03/

# comments and blank lines are allowed
rm s3://my-bucket/2020/03/19/file2.gz
mv s3://my-bucket/old/file.gz s3://my-bucket/archive/file.gz
```

### 6. Target an S3-compatible service with `--endpoint-url`

`s5cmd` works with any S3-API-compatible storage — Google Cloud Storage, MinIO, Cloudflare R2, and others — by pointing `--endpoint-url` at the service:

```bash
# List buckets on Google Cloud Storage
s5cmd --endpoint-url https://storage.googleapis.com ls

# Same thing via an environment variable
export S3_ENDPOINT_URL="https://storage.googleapis.com"
s5cmd ls

# Copy to a MinIO instance
s5cmd --endpoint-url http://localhost:9000 cp report.csv s3://my-bucket/
```

> For GCS, populate `aws_access_key_id` and `aws_secret_access_key` in `~/.aws/credentials` with an [HMAC key](https://cloud.google.com/storage/docs/authentication/managing-hmackeys#create).

## Worth Knowing

A few extras that round out the tool: `sync` for one-way synchronization with `--delete` support, `--include` / `--exclude` filters for pattern matching, and `--dry-run` to preview any destructive operation before committing to it. Note that `s5cmd` does not aim for command compatibility with `aws-cli`, so don't assume flags map across one-to-one.

For most workflows the payoff is immediate: the same operations you already run, finished in a fraction of the time.
