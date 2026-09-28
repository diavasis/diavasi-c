# C client

[![CI](https://github.com/diavasis/diavasi-c/actions/workflows/ci.yml/badge.svg)](https://github.com/diavasis/diavasi-c/actions/workflows/ci.yml)
[![tag](https://img.shields.io/github/v/tag/diavasis/diavasi-c)](https://github.com/diavasis/diavasi-c/tags)
[![license](https://img.shields.io/github/license/diavasis/diavasi-c)](https://github.com/diavasis/diavasi-c/blob/main/LICENSE)

`diavasi_consume` is a thin client of `diavasi.data.v1` on the gRPC C stack. It opens a TLS stream, sends the bearer token, Hello version 1, then JoinGroup, and acks each batch. The caller stores no cursor and does not dedupe on `record_id`. A dropped stream is how unacked batches return. Reconnect with the same consumer id and the server replays them.

`proto/data.proto` in this repository is the copy of `diavasi.data.v1` from [github.com/diavasis/diavasi](https://github.com/diavasis/diavasi) tag `v0.13.0`. There is no package registry. Version 0.1.0 is the git tag `v0.1.0` of this repository. Zig calls this library from [diavasi-zig](https://github.com/diavasis/diavasi-zig).

## Install

`make test-proto` needs a C11 compiler. It checks the frame codec and does not link gRPC.

`diavasi_consume`, `process`, and `consume_test` link the gRPC C core. `pkg-config` must resolve the `grpc` module. If it does not, `cc` stops with `grpc/grpc.h` file not found.

Debian and Ubuntu:

```bash
sudo apt-get install build-essential pkg-config libgrpc-dev
```

macOS, with Homebrew:

```bash
brew install grpc
```

Homebrew installs `grpc.pc` where its `pkg-config` already looks (`/opt/homebrew/lib/pkgconfig` on Apple silicon). Confirm the module, then build:

```bash
pkg-config --exists grpc && echo ok
make
```

`make` writes `./diavasi_consume` and `./process` in this directory.

## Library

```c
#include "diavasi.h"

static void on_record(uint64_t batch_id, uint64_t record_id,
                      const uint8_t *payload, size_t payload_len, void *user) {
    (void)payload;
    (void)user;
    printf("batch %llu record %llu (%zu bytes)\n",
           (unsigned long long)batch_id,
           (unsigned long long)record_id,
           payload_len);
}

diavasi_options options = {
    .addr = "127.0.0.1:7710",
    .ca_path = "/tmp/diavasi-sdk/dataplane-ca.crt",
    .token = "sdk-demo",
    .group_id = "demo",
    .consumer_id = "c",
    .max_in_flight = 1,
    .expect_records = 8,
    .on_record = on_record,
};
diavasi_report report;
char error[256] = {0};
int rc = diavasi_consume(&options, &report, error, sizeof error);
if (rc >= 1 && rc <= 8) {
    fprintf(stderr, "protocol %d: %s\n", rc, error);
} else if (rc != 0) {
    fprintf(stderr, "%s\n", error);
}
diavasi_report_free(&report);
```

`on_record` runs before the batch is acked. `payload` is valid only for that call. `max_in_flight` defaults to 1 when left 0. `halt_after_acks` closes after that many acks and does not send Leave. `expect_records` sends Leave once that many records are acked.

Return 0 on success. Protocol codes 1 through 8 are returned as that integer: bad version, bad state, unknown ack, duplicate ack, group not running, unsupported, internal, heartbeat timeout. Any other failure returns -1. `error` receives the message. A bad token sets `grpc UNAUTHENTICATED: unauthorized`. A group that is not running sets `protocol error 5`.

## Run

Start the server from the repo root:

```bash
cargo build -p diavasi
export PATH="$PWD/target/debug:$PATH"
mkdir -p /tmp/diavasi-sdk
diavasi serve --bind 127.0.0.1:7700 --data-bind 127.0.0.1:7710 \
  --store /tmp/diavasi-sdk/state --token sdk-demo
```

In a second terminal, from the repo root:

```bash
curl -fsS -X POST -H "Authorization: Bearer sdk-demo" \
  http://127.0.0.1:7700/v1/groups/demo/pause || true
curl -fsS -X DELETE -H "Authorization: Bearer sdk-demo" \
  http://127.0.0.1:7700/v1/groups/demo || true
curl -fsS -H "Authorization: Bearer sdk-demo" -H "content-type: application/json" \
  -d '{"group_id":"demo","total_records":8,"payload_size":8,"max_buffer_records":64,"max_buffer_bytes":65536,"batch_max_records":4,"batch_timeout_ms":200,"ordering_contract":"synthetic-u64"}' \
  http://127.0.0.1:7700/v1/groups
curl -fsS -X POST -H "Authorization: Bearer sdk-demo" \
  http://127.0.0.1:7700/v1/groups/demo/start

make process
./process
```

`examples/process.c` is that program. `make` also builds `./diavasi_consume`, the flag client used by the compatibility suite:

```bash
./diavasi_consume --addr 127.0.0.1:7710 --ca /tmp/diavasi-sdk/dataplane-ca.crt \
  --token sdk-demo --group demo --consumer c --total 8
```

Flags: `--addr`, `--ca`, `--token`, `--group`, `--consumer`, `--total`, `--max-in-flight` (default 1), `--halt-after`. The last occurrence of a flag wins. The program prints `record_ids` and `batch_ids`.

Delete returns 409 while the group is running, so the Run commands pause it first. `POST .../start` on a group that already exists resumes its cursor. After `demo` has delivered all 8 records, this command stays on the stream, receives only heartbeats, and prints nothing. Pause, delete, create, and start the group before running it again.

## Test

`make test-proto` checks the frame codec and does not need a server or gRPC. `make test` runs that, then builds and runs `consume_test` when `pkg-config --exists grpc` succeeds. Without the module it prints `skip: grpc headers are not installed` and still exits 0.

`consume_test` exits 0 without contacting a server until `DIAVASI_DATA_ADDR`, `DIAVASI_CA`, and `DIAVASI_API_TOKEN` are set. With a running server and a synthetic group:

```bash
export DIAVASI_DATA_ADDR=127.0.0.1:7710
export DIAVASI_CA=/tmp/diavasi-sdk/dataplane-ca.crt
export DIAVASI_API_TOKEN=sdk-demo
export DIAVASI_GROUP=sdk
export DIAVASI_TOTAL=8
make test
```

The test joins that group as consumer `c-test` and fails unless it receives `DIAVASI_TOTAL` records (default 8). A group that has already delivered those records must be paused, deleted, created, and started again. Starting it only resumes the cursor. The server commands are in [Run](#run).
