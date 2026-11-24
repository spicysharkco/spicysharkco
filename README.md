<p align="center">
  <strong>Spicy Shark</strong><br>
  AI research and infrastructure engineering: reliability and performance.<br>
  Go · Rust · Python · C++ · TypeScript
</p>

<p align="center">
  <a href="https://spicyshark.co"><img alt="Site" src="https://img.shields.io/badge/Site-spicyshark.co-0a1626?style=for-the-badge"></a>
  <a href="mailto:hello@spicyshark.co"><img alt="Email" src="https://img.shields.io/badge/Email-hello%40spicyshark.co-0a1626?style=for-the-badge"></a>
  <a href="https://x.com/spicysharkco"><img alt="X" src="https://img.shields.io/badge/X-spicysharkco-0a1626?style=for-the-badge"></a>
  <a href="https://huggingface.co/spicysharkco"><img alt="Hugging Face" src="https://img.shields.io/badge/Hugging%20Face-spicysharkco-0a1626?style=for-the-badge"></a>
</p>

I find and fix performance and reliability bugs in the infrastructure AI runs on, and write up how each one was found.

More at [Learn](https://spicyshark.co/learn/), [Research](https://spicyshark.co/research/) and the [open-source record](https://spicyshark.co/open-source/).

<p align="center">
  <a href="#selected-fixes">Selected fixes</a> · <a href="#published-security-advisories">Security advisories</a> · <a href="#my-own-projects">My projects</a> · <a href="#research-in-progress">Research in progress</a> · <a href="#upstream-work">Upstream work</a>
</p>

## Selected fixes

| Project | Bug | Result | Merged |
|---|---|---|---|
| [kube-router](https://github.com/cloudnativelabs/kube-router/pull/2192) | Network-policy sync copied and scanned the whole iptables-save buffer once per local pod | About 320x faster and 99.8% less allocation per op (benchmark: 80 pods, 1 MiB of existing rules) | 2026-10-12 |
| [k3s](https://github.com/k3s-io/k3s/pull/14811) | Goroutine leak and data race in the server's tunnel proxy | On multi-core runners the old code left 1029 goroutines behind in the new test; the fix passes `go test -race -count=20` 20 of 20 | 2026-10-15 |
| [OpenCV](https://github.com/opencv/opencv/pull/30316) | Out-of-bounds access in `validateDisparity`, the stereo-depth left-right check | AddressSanitizer: all 4 new test cases overflow without the fix, all 4 pass with it | 2026-10-16 |
| [llm-d-router](https://github.com/llm-d/llm-d-router/pull/3476) | Prefix-cache index sized in engine blocks instead of router blocks | With vLLM's default block size the index could track 4x the tokens the engine's KV cache held (up to 64x at block size 1); 3 of 6 new test cases failed on main, all 6 pass with the fix | 2026-10-20 |
| [Litestream](https://github.com/benbjohnson/litestream/pull/1576) | WebDAV replicas dated each backup file by its upload time, not its header timestamp, so timestamp-based restore could pick the wrong file | Three existing timestamp tests against a WebDAV server fail without the fix (one off by almost an hour) and pass with it | 2026-10-27 |
| [Text Embeddings Inference](https://github.com/huggingface/text-embeddings-inference/pull/952) | Concurrent `/health` checks during a long forward pass filled the backend queue, so the next embed call panicked and the server process exited | Load test with forward passes slowed on purpose: without the fix the server was gone by 11 s; with it the server ran the whole 75 s and nothing panicked | 2026-11-03 |
| [urllib3](https://github.com/urllib3/urllib3/pull/5316) | `HTTPResponse.read(amt)` skipped the last decoder flush at the end of the body and dropped the tail of raw deflate responses | A raw deflate body that decodes to 6000 bytes, read 64 bytes at a time, came back as 5952 bytes; new tests for raw deflate, `deflate, gzip` and truncated zstd bodies fail without the fix and pass with it | 2026-11-11 |

Each fix in full: the failure, the fix, the result and why it matters.

<details>
<summary><b>kube-router</b>: network-policy sync 320x faster, 99.8% less allocation (benchmark)</summary>

- Failure: `syncPodFirewallChains` copied and scanned the whole iptables-save buffer once per local pod and IP family.
- Fix: remove the check. The rule text it looked for is unique to one pod in one sync, so it could never match.
- Result: `BenchmarkSyncPodFirewallChains` (80 pods, one family, 1 MiB of existing rules, median of 5 runs) went from 152,895,797 to 477,097 ns/op, about 320x, and from 90.5 MB to 0.2 MB allocated per op. Validation was unit tests and the benchmark; no Kubernetes integration test.
- Why it matters: kube-router is a pod network, firewall and service proxy for Kubernetes, and its network-policy controller runs this sync on every node. On a node with 84 pods the removed copy cost about 175 ms of CPU and 100 MB of allocation per full sync ([issue #2191](https://github.com/cloudnativelabs/kube-router/issues/2191)).
- [PR #2192](https://github.com/cloudnativelabs/kube-router/pull/2192), merged 2026-10-12 · [post](https://spicyshark.co/2026/10/15/kube-router-netpol-sync-copy.html)

</details>

<details>
<summary><b>k3s</b>: a goroutine leak in the proxy</summary>

- Failure: when both directions of a connection failed together, as on a TCP reset, the second goroutine blocked forever on an unbuffered error channel. It held a 32 KiB buffer and both connections. A plain bool flag was written by both goroutines.
- Fix: the channel has room for both errors, and the flag is gone, which removes the data race.
- Result: a new test runs `Proxy` 10,000 times on connections that always fail. The old code left 1029 goroutines behind and `-race` reported the race. With the fix, `go test -race -count=20` passed 20 of 20. The test catches the bug on multi-core runners; with `GOMAXPROCS=1` the old code can pass.
- Why it matters: the leak is in the server's tunnel proxy, which the default egress-selector mode uses. On a server whose tunnel connections end in resets, for example traffic to an aggregated API service, the leaked goroutines only grow until a restart, each holding a 32 KiB buffer and two connections.
- [PR #14811](https://github.com/k3s-io/k3s/pull/14811), merged 2026-10-15 · [post](https://spicyshark.co/2026/10/16/k3s-proxy-goroutine-leak.html)

</details>

<details>
<summary><b>OpenCV</b>: out-of-bounds access in <code>validateDisparity</code></summary>

- Failure: a disparity outside the valid range made the computed index `x2` negative or at least `cols`, and the code used it to index a buffer unchecked, in both the `CV_16S` and `CV_32S` branches.
- Fix: skip such pixels in both branches with one unsigned bounds check.
- Result: the new test has four cases. AddressSanitizer reports a heap-buffer-overflow in all four without the fix; all four pass with it. The 5.x port is [#30344](https://github.com/opencv/opencv/pull/30344).
- Why it matters: `validateDisparity` is the left-right check in stereo depth. Given a disparity map computed with other parameters, it wrote 32 bytes before its buffer. Without a sanitizer that left a bad pixel marked valid: a wrong depth value with no error.
- [PR #30316](https://github.com/opencv/opencv/pull/30316), merged 2026-10-16 · [post](https://spicyshark.co/2026/10/16/opencv-validatedisparity-out-of-bounds.html)

</details>

<details>
<summary><b>llm-d-router</b>: the prefix-cache index sized in the right units</summary>

- Failure: with `autoTune` on, the approximate prefix cache sized each pod's LRU from the engine's block count. Each LRU entry is one router block of at least 64 tokens, and vLLM's default engine block is 16 tokens. So the index could track four times as many tokens as the engine's KV cache held, and up to 64 times with a block size of 1.
- Fix: convert the capacity to router blocks, `CacheNumBlocks * CacheBlockSize / GetBlockSize()`, rounded down and at least 1. It is unchanged when the two block sizes match or the engine does not report its block size.
- Result: a new table test has six cases. On main three of them failed (31250 entries instead of 7812, 2 instead of 1, 1000 instead of 64000); with the fix all six pass under `go test -race`.
- Why it matters: llm-d-router picks the pod for each inference request, partly by how much of the prompt each pod already has cached. An index larger than the engine's cache keeps prefixes the engine has evicted, so the prefix-cache scorer can overestimate cache hits and send a request to a pod that no longer holds its prefix. The analysis and reproduction in [issue #3139](https://github.com/llm-d/llm-d-router/issues/3139) pointed at the cause.
- [PR #3476](https://github.com/llm-d/llm-d-router/pull/3476), merged 2026-10-20 · [post](https://spicyshark.co/2026/11/23/llm-d-router-autotuned-lru-router-blocks.html)

</details>

<details>
<summary><b>Litestream</b>: restore by timestamp over WebDAV read the wrong clock</summary>

- Failure: `webdav.ReplicaClient.LTXFiles` ignored its `useMetadata` option and reported the server's modification time as `CreatedAt`. That is when the server received the file, not the timestamp in its LTX header (the header of Litestream's backup file format), so timestamp-based restore could pick the wrong file or find none. Like the jobs fixed in [#1575](https://github.com/benbjohnson/litestream/pull/1575), the WebDAV CI job ran only the file client, so the shared integration suite never ran against a WebDAV server in CI.
- Fix: with `useMetadata` set, the client reads each listed file's header with one ranged GET, 10 at a time, and uses its timestamp. A file whose header cannot be read fails the listing with an error. The WebDAV CI job now runs the WebDAV client.
- Result: against the CI job's WebDAV server, three existing timestamp tests in the shared integration suite fail without the fix and pass with it; `TestReplicaClient_TimestampPreservation` was off by almost an hour (59m59.647s). New unit tests in the `webdav` package fail without the change, and `go test -race ./webdav/` passes with it. The cost: on that test server, listing 999 files took 0.2 to 0.4 s with header reads on, about 0.05 s with them off.
- Why it matters: Litestream is a disaster recovery tool for SQLite that copies each change to another location, such as a file, object storage or a WebDAV server. Restore to a timestamp is its point-in-time recovery, and on WebDAV it could choose files by the wrong clock.
- [PR #1576](https://github.com/benbjohnson/litestream/pull/1576), merged 2026-10-27 · [post](https://spicyshark.co/2026/11/23/litestream-webdav-ltx-header-timestamps.html)

</details>

<details>
<summary><b>Text Embeddings Inference</b>: health checks could abort the server</summary>

- Failure: `Backend::embed` and `Backend::predict` handed work to the backend thread with `try_send(..).expect(..)` on a channel of 8 slots that `/health` checks share. During a long forward pass, concurrent health checks fill those slots; `try_send` then returns `Full` and the `expect` panics. The release profile sets `panic = "abort"`, so the whole server exits. It is the panic site of [#424](https://github.com/huggingface/text-embeddings-inference/issues/424); the reproduction there needed memory pressure, this one does not.
- Fix: both calls use `send(..).await`, as `Backend::health` already did. A full queue now makes the caller wait instead of ending the process.
- Result: a load test sent batches of 32 inputs to `/embed` from 4 threads and called `/health` from 32 threads, with forward passes slowed on purpose (compute capped at 4 CPU threads). Without the fix the server was gone by 11 s, with the `Full(..)` panic in its log; the same embed load without health traffic ran 40 s without a panic. With the fix the server ran the whole 75 s: 9 embed requests and 2334 health checks, all 200, and no panic. A new unit test fills the queue behind a blocked forward pass and fails with the same panic without the fix. Limit: only the CPU build was run end to end, and `/health` still waits behind forward passes, so a short probe timeout can still fail.
- Why it matters: Text Embeddings Inference is Hugging Face's server for embedding and reranker models. Health checks during a long forward pass were enough to crash a busy server; no memory pressure or other fault was needed.
- [PR #952](https://github.com/huggingface/text-embeddings-inference/pull/952), merged 2026-11-03 · [post](https://spicyshark.co/2026/11/16/text-embeddings-inference-health-checks-full-backend-queue.html)

</details>

<details>
<summary><b>urllib3</b>: reads in small chunks dropped the end of a raw deflate body</summary>

- Failure: at the end of the body, `read(amt)` returned early and skipped the last `_flush_decoder()`. Since decoding became bounded with the decompression-bomb safeguards, zlib can still hold output when a read stops at exactly `amt` bytes. zlib and gzip bodies end in a checksum that is still unread then, but raw deflate has no trailer, so of complete bodies only raw deflate lost data: `b"abc" * 2000` read with `while resp.read(64)` came back as 5952 of 6000 bytes, and `stream(64)` did the same. Skipping the flush also let a truncated zstd body read with `read(amt)` end short with no `DecodeError`.
- Fix: the early return no longer skips the flush once the decoder has decoded data; `stream()` keeps reading until that flushing read; and the loop that fills a read up to `amt` checks for the end of the body after each raw read. A truncated zstd body read with `read(amt)` now raises `DecodeError`, as `read()` and `read1()` already did.
- Result: three of the new tests, small reads of a raw deflate body, of a `deflate, gzip` body and of a truncated zstd body, fail without the fix and pass with it. A companion fix, [#5315](https://github.com/urllib3/urllib3/pull/5315), covers gzip bodies of more than one member when a read ends exactly at a member boundary: with two 100-byte members, `while resp.read(100)` returned the second member over and over and never reached the end.
- Why it matters: urllib3 is the HTTP client under `requests`. One bug returned a shorter body with no error, for raw deflate bodies read in chunks; the other could make a read loop run forever when a chunk ended on a gzip member boundary.
- [PR #5316](https://github.com/urllib3/urllib3/pull/5316), merged 2026-11-11 · [post](https://spicyshark.co/2026/11/16/urllib3-deflate-flush-bounded-read.html)

</details>

## Published security advisories

GitHub Security Advisories that list me in their credits. The Credit column gives the role the advisory records.

| Project | Advisory | CVE | Severity | Credit | Published |
|---|---|---|---|---|---|
| [LMCache](https://github.com/LMCache/LMCache) | [GHSA-hhqh-8jpj-27rf](https://github.com/LMCache/LMCache/security/advisories/GHSA-hhqh-8jpj-27rf): the frontend proxy forwarded unauthenticated requests to any local UNIX socket · [post](https://spicyshark.co/2026/11/24/lmcache-frontend-proxy-could-reach-any-local-unix-socket-fixed-in-0-5-8.html) |  | Critical 9.8 | Finder | 2026-11-23 |

## My own projects

- [sharkline](https://github.com/spicysharkco/sharkline): a status line for Claude Code, in Rust. A warm render takes about 1.3 ms with no child process and no network; a Node process that does nothing takes about 30 ms on the same machine. Try: `cargo install --locked --git https://github.com/spicysharkco/sharkline`.
- [sharkswap](https://github.com/spicysharkco/sharkswap): a Claude Code account switcher in Rust. It swaps between saved accounts, shows each account's 5-hour and weekly usage, and can move the login when one runs out. One binary, no daemon. Try: `cargo install --locked --git https://github.com/spicysharkco/sharkswap`, then `sharkswap probe`.
- [chess-sim2real-benchmark](https://github.com/spicysharkco/chess-sim2real-benchmark): a render generator and a 20,000-image benchmark for chess-position recognition, with [a paper on what synthetic training data buys and costs](https://github.com/spicysharkco/chess-sim2real-benchmark/blob/main/paper/main.pdf) ([dataset](https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark) <a href="https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark"><img alt="Hugging Face downloads" src="https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fhuggingface.co%2Fapi%2Fdatasets%2Fspicysharkco%2Fchess-sim2real-benchmark%3Fexpand%255B%255D%3DdownloadsAllTime&query=%24.downloadsAllTime&label=Hugging%20Face%20downloads&color=0a1626&style=flat-square"></a>). The README walks through rendering and scoring four frozen views locally.

## Research in progress

- **Quantization.** GGUF files that give each tensor group its own quant type, chosen by measured sensitivity, inside a fixed file size ([model card](https://huggingface.co/spicysharkco/Qwen3.8-27B-OBLITERATED-DYN-GGUF)). At the same size as the upstream Q4_K_M (16.81 GB), mean KL divergence from BF16 fell from 0.0348 to 0.0195 on 30,690 held-out tokens. About two thirds of that gain comes from the importance matrix; the rest comes from the per-tensor mix. CPU prompt processing runs at 0.85x the upstream file's speed. Limit: the evaluation is text only, and no task benchmark was run.
- **Chess benchmark.** Changing which corner is light in the renderer moved per-square accuracy from 54.4% to 85.2%, and only 18 of 192 boards are exact after the fix. Limit: the public reproduction covers a bounded render and scoring path, not all 20,000 cells or the adaptation-training study.

## Upstream work

Merged fixes, grouped by area. Each name links to its merged pull requests.

- **Kubernetes and cloud native:** [k3s](https://github.com/k3s-io/k3s/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [kube-router](https://github.com/cloudnativelabs/kube-router/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Cluster API](https://github.com/kubernetes-sigs/cluster-api/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [external-dns](https://github.com/kubernetes-sigs/external-dns/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [cert-manager](https://github.com/cert-manager/cert-manager/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Linkerd](https://github.com/linkerd/linkerd2/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Dapr](https://github.com/dapr/dapr/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Velero](https://github.com/velero-io/velero/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [CNI plugins](https://github.com/containernetworking/plugins/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [kube-vip](https://github.com/kube-vip/kube-vip/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [stern](https://github.com/stern/stern/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco)
- **AI and ML:** [AIBrix (vLLM project)](https://github.com/vllm-project/aibrix/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [llm-d Router](https://github.com/llm-d/llm-d-router/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Kueue](https://github.com/kubernetes-sigs/kueue/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [KubeAI](https://github.com/kubeai-project/kubeai/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Kubeflow Trainer](https://github.com/kubeflow/trainer/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Hugging Face Text Embeddings Inference](https://github.com/huggingface/text-embeddings-inference/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [ONNX](https://github.com/onnx/onnx/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [OpenCV](https://github.com/opencv/opencv/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Feast](https://github.com/feast-dev/feast/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco)
- **Clients and observability:** [redis-py](https://github.com/redis/redis-py/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [go-redis](https://github.com/redis/go-redis/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Prometheus Go client](https://github.com/prometheus/client_golang/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [urllib3](https://github.com/urllib3/urllib3/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [NATS Python client](https://github.com/nats-io/nats.py/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco)
- **Developer, data and security tools:** [bat](https://github.com/sharkdp/bat/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Grype](https://github.com/anchore/grype/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Litestream](https://github.com/benbjohnson/litestream/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco)

The full list, with write-ups: [spicyshark.co/open-source](https://spicyshark.co/open-source/).
