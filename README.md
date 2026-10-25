<p align="center">
  <strong>Hi, I'm spicysharkco.</strong><br>
  AI research and infrastructure engineering: reliability and performance.<br>
  Go · Rust · Python · C++ · TypeScript
</p>

<p align="center">
  <a href="https://spicyshark.co"><img alt="Site" src="https://img.shields.io/badge/Site-spicyshark.co-0a1626?style=for-the-badge"></a>
  <a href="mailto:hello@spicyshark.co"><img alt="Email" src="https://img.shields.io/badge/Email-hello%40spicyshark.co-0a1626?style=for-the-badge"></a>
  <a href="https://x.com/spicysharkco"><img alt="X" src="https://img.shields.io/badge/X-spicysharkco-0a1626?style=for-the-badge"></a>
  <a href="https://huggingface.co/spicysharkco"><img alt="Hugging Face" src="https://img.shields.io/badge/Hugging%20Face-spicysharkco-0a1626?style=for-the-badge"></a>
</p>

Making the AI stack faster and harder to break, one merged fix at a time. Receipts below.

More at [Learn](https://spicyshark.co/learn/), [Research](https://spicyshark.co/research/) and the [open-source record](https://spicyshark.co/open-source/).

## Selected fixes

### kube-router: network-policy sync 320x faster, 99.8% less allocation (benchmark)

- Failure: `syncPodFirewallChains` copied and scanned the whole iptables-save buffer once per local pod and IP family.
- Fix: remove the check. The rule text it looked for is unique to one pod in one sync, so it could never match.
- Result: `BenchmarkSyncPodFirewallChains` (80 pods, one family, 1 MiB of existing rules, median of 5 runs) went from 152,895,797 to 477,097 ns/op, about 320x, and from 90.5 MB to 0.2 MB allocated per op. Validation was unit tests and the benchmark; no Kubernetes integration test.
- Why it matters: kube-router is a pod network, firewall and service proxy for Kubernetes, and its network-policy controller runs this sync on every node. On a node with 84 pods the removed copy cost about 175 ms of CPU and 100 MB of allocation per full sync ([issue #2191](https://github.com/cloudnativelabs/kube-router/issues/2191)).
- [PR #2192](https://github.com/cloudnativelabs/kube-router/pull/2192), merged 2026-10-12 · [post](https://spicyshark.co/2026/10/15/kube-router-netpol-sync-copy.html)

### k3s: a goroutine leak in the proxy

- Failure: when both directions of a connection failed together, as on a TCP reset, the second goroutine blocked forever on an unbuffered error channel. It held a 32 KiB buffer and both connections. A plain bool flag was written by both goroutines.
- Fix: the channel has room for both errors, and the flag is gone, which removes the data race.
- Result: a new test runs `Proxy` 10,000 times on connections that always fail. The old code left 1029 goroutines behind and `-race` reported the race. With the fix, `go test -race -count=20` passed 20 of 20. The test catches the bug on multi-core runners; with `GOMAXPROCS=1` the old code can pass.
- Why it matters: the leak is in the server's tunnel proxy, which the default egress-selector mode uses. On a server whose tunnel connections end in resets, for example traffic to an aggregated API service, the leaked goroutines only grow until a restart, each holding a 32 KiB buffer and two connections.
- [PR #14811](https://github.com/k3s-io/k3s/pull/14811), merged 2026-10-15 · [post](https://spicyshark.co/2026/10/16/k3s-proxy-goroutine-leak.html)

### OpenCV: out-of-bounds access in `validateDisparity`

- Failure: a disparity outside the valid range made the computed index `x2` negative or at least `cols`, and the code used it to index a buffer unchecked, in both the `CV_16S` and `CV_32S` branches.
- Fix: skip such pixels in both branches with one unsigned bounds check.
- Result: the new test has four cases. AddressSanitizer reports a heap-buffer-overflow in all four without the fix; all four pass with it. The 5.x port is [#30344](https://github.com/opencv/opencv/pull/30344).
- Why it matters: `validateDisparity` is the left-right check in stereo depth. Given a disparity map computed with other parameters, it wrote 32 bytes before its buffer. Without a sanitizer that left a bad pixel marked valid: a wrong depth value with no error.
- [PR #30316](https://github.com/opencv/opencv/pull/30316), merged 2026-10-16 · [post](https://spicyshark.co/2026/10/16/opencv-validatedisparity-out-of-bounds.html)

### llm-d-router: the prefix-cache index sized in the right units

- Failure: with `autoTune` on, the approximate prefix cache sized each pod's LRU from the engine's block count. Each LRU entry is one router block of at least 64 tokens, and vLLM's default engine block is 16 tokens. So the index could track four times as many tokens as the engine's KV cache held, and up to 64 times with a block size of 1.
- Fix: convert the capacity to router blocks, `CacheNumBlocks * CacheBlockSize / GetBlockSize()`, rounded down and at least 1. It is unchanged when the two block sizes match or the engine does not report its block size.
- Result: a new table test has six cases. On main three of them failed (31250 entries instead of 7812, 2 instead of 1, 1000 instead of 64000); with the fix all six pass under `go test -race`.
- Why it matters: llm-d-router picks the pod for each inference request, partly by how much of the prompt each pod already has cached. An index larger than the engine's cache keeps prefixes the engine has evicted, so the prefix-cache scorer can overestimate cache hits and send a request to a pod that no longer holds its prefix. The analysis and reproduction in [issue #3139](https://github.com/llm-d/llm-d-router/issues/3139) pointed at the cause.
- [PR #3476](https://github.com/llm-d/llm-d-router/pull/3476), merged 2026-10-20

## My own projects

- [sharkline](https://github.com/spicysharkco/sharkline): a status line for Claude Code, in Rust. A warm render takes about 1.3 ms with no child process and no network; a Node process that does nothing takes about 30 ms on the same machine. Try: `cargo install --locked --git https://github.com/spicysharkco/sharkline`.
- [sharkswap](https://github.com/spicysharkco/sharkswap): a Claude Code account switcher in Rust. It swaps between saved accounts, shows each account's 5-hour and weekly usage, and can move the login when one runs out. One binary, no daemon. Try: `cargo install --locked --git https://github.com/spicysharkco/sharkswap`, then `sharkswap probe`.
- [chess-sim2real-benchmark](https://github.com/spicysharkco/chess-sim2real-benchmark): a render generator and a 20,000-image benchmark for chess-position recognition ([dataset](https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark)). The README walks through rendering and scoring four frozen views locally.

## Research in progress

- **Quantization.** GGUF files that give each tensor group its own quant type, chosen by measured sensitivity, inside a fixed file size ([model card](https://huggingface.co/spicysharkco/Qwen3.8-27B-OBLITERATED-DYN-GGUF)). At the same size as the upstream Q4_K_M (16.81 GB), mean KL divergence from BF16 fell from 0.0348 to 0.0195 on 30,690 held-out tokens. About two thirds of that gain comes from the importance matrix; the rest comes from the per-tensor mix. CPU prompt processing runs at 0.85x the upstream file's speed. Limit: the evaluation is text only, and no task benchmark was run.
- **Chess benchmark.** Changing which corner is light in the renderer moved per-square accuracy from 54.4% to 85.2%, and only 18 of 192 boards are exact after the fix. Limit: the public reproduction covers a bounded render and scoring path, not all 20,000 cells or the adaptation-training study.

## Contributions

<p align="center">
  <a href="https://github.com/cloudnativelabs/kube-router"><img src="https://github.com/cloudnativelabs.png?size=64" width="48" height="48" alt="kube-router" title="kube-router"></a>
  <a href="https://github.com/k3s-io/k3s"><img src="https://github.com/k3s-io.png?size=64" width="48" height="48" alt="k3s" title="k3s"></a>
  <a href="https://github.com/opencv/opencv"><img src="https://github.com/opencv.png?size=64" width="48" height="48" alt="OpenCV" title="OpenCV"></a>
  <a href="https://github.com/kubernetes-sigs/external-dns"><img src="https://github.com/kubernetes-sigs.png?size=64" width="48" height="48" alt="external-dns" title="external-dns"></a>
  <a href="https://github.com/vllm-project/aibrix"><img src="https://github.com/vllm-project.png?size=64" width="48" height="48" alt="AIBrix" title="AIBrix"></a>
  <a href="https://github.com/ome-projects/ome"><img src="https://github.com/ome-projects.png?size=64" width="48" height="48" alt="OME" title="OME"></a>
  <a href="https://github.com/bluenviron/mediamtx"><img src="https://github.com/bluenviron.png?size=64" width="48" height="48" alt="mediamtx" title="mediamtx"></a>
  <a href="https://github.com/clastix/kamaji"><img src="https://github.com/clastix.png?size=64" width="48" height="48" alt="Kamaji" title="Kamaji"></a>
  <a href="https://github.com/DNSControl/dnscontrol"><img src="https://github.com/DNSControl.png?size=64" width="48" height="48" alt="DNSControl" title="DNSControl"></a>
  <a href="https://github.com/derailed/k9s"><img src="https://github.com/derailed.png?size=64" width="48" height="48" alt="k9s" title="k9s"></a>
  <a href="https://github.com/minio/minio-go"><img src="https://github.com/minio.png?size=64" width="48" height="48" alt="minio-go" title="minio-go"></a>
  <a href="https://github.com/dapr/dapr"><img src="https://github.com/dapr.png?size=64" width="48" height="48" alt="Dapr" title="Dapr"></a>
</p>

| Project | What it is | Merged PRs | GitHub stars |
|---|---|---|---|
| [OpenCV](https://github.com/opencv/opencv) | Computer vision library | 2 | 91.3k |
| [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) | Agent orchestration for OpenCode | 2 | 70.2k |
| [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) | Multi-agent orchestration for Claude Code | 7 | 40.1k |
| [k9s](https://github.com/derailed/k9s) | Terminal UI for Kubernetes clusters | 1 | 34.9k |
| [k3s](https://github.com/k3s-io/k3s) | Lightweight Kubernetes | 2 | 34.2k |
| [oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex) | Hooks and agent teams for Codex CLI | 4 | 33.6k |
| [Dapr](https://github.com/dapr/dapr) | Runtime for distributed applications | 1 | 26.2k |
| [yfinance](https://github.com/ranaroussi/yfinance) | Market data from Yahoo Finance in Python | 1 | 25.6k |
| [Prefect](https://github.com/PrefectHQ/prefect) | Workflow orchestration for data pipelines in Python | 1 | 24.0k |
| [ONNX](https://github.com/onnx/onnx) | Open standard for machine learning models | 1 | 21.6k |
| [mediamtx](https://github.com/bluenviron/mediamtx) | Live media server (RTSP, WebRTC, SRT and more) | 7 | 20.5k |
| [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) | gRPC to JSON proxy generator | 1 | 20.0k |
| [Litestream](https://github.com/benbjohnson/litestream) | Streaming replication for SQLite | 1 | 14.5k |
| [pgx](https://github.com/jackc/pgx) | PostgreSQL driver for Go | 1 | 14.3k |
| [cert-manager](https://github.com/cert-manager/cert-manager) | TLS certificate automation for Kubernetes | 1 | 14.1k |
| [redis-py](https://github.com/redis/redis-py) | Python client for Redis | 2 | 13.7k |
| [Kubescape](https://github.com/kubescape/kubescape) | Kubernetes security scanner | 2 | 11.8k |
| [PCL](https://github.com/PointCloudLibrary/pcl) | 3D point cloud processing in C++ | 1 | 11.2k |
| [Xinference](https://github.com/xorbitsai/inference) | Serving for open LLMs, speech and multimodal models | 6 | 9.6k |
| [goja](https://github.com/dop251/goja) | JavaScript engine in pure Go | 11 | 7.2k |
| [AIBrix](https://github.com/vllm-project/aibrix) | Infrastructure for LLM inference on Kubernetes (vLLM project) | 13 | 5.2k |
| [DNSControl](https://github.com/DNSControl/dnscontrol) | DNS configuration as code | 8 | 4.0k |
| [NumPyro](https://github.com/pyro-ppl/numpyro) | Probabilistic programming on JAX | 5 | 2.8k |
| [kube-router](https://github.com/cloudnativelabs/kube-router) | Pod network, firewall and service proxy for Kubernetes | 2 | 2.5k |
| [Kamaji](https://github.com/clastix/kamaji) | Hosted Kubernetes control planes | 13 | 2.1k |
| [OjG](https://github.com/ohler55/ojg) | Fast JSON for Go | 25 | 956 |
| [sql_exporter](https://github.com/burningalchemist/sql_exporter) | SQL metrics for Prometheus | 20 | 601 |
| [oras-go](https://github.com/oras-project/oras-go) | Go library for OCI artifacts | 17 | 295 |
| [Soup Sieve](https://github.com/facelessuser/soupsieve) | CSS selectors for Beautiful Soup | 8 | 271 |
| [wcmatch](https://github.com/facelessuser/wcmatch) | Wildcard and glob matching for Python | 7 | 169 |

Star counts as of 2026-10-20.

## Recent merges

Newest first, one per project.

### AI and LLM inference

| Project | Change | Result | Date |
|---|---|---|---|
| [llmfit](https://github.com/AlexsJones/llmfit) | [#1169](https://github.com/AlexsJones/llmfit/pull/1169) usable context under MoE offload charges only the active experts held in VRAM | A 6 GiB GPU rated Qwen3-30B-A3B at Q8_0 Good with no room left for context (`40k→0`); now `40k→25k` | 2026-10-25 |
| [ONNX](https://github.com/onnx/onnx) | [#8629](https://github.com/onnx/onnx/pull/8629) the reference `Shape` clamps an `end` below `-rank` to an empty result, as the spec says | `Shape(end=-4)` on a 2x3x4 input gave `[2 3]`; now `[]` | 2026-10-25 |
| [Kubeflow Trainer](https://github.com/kubeflow/trainer) | [#4255](https://github.com/kubeflow/trainer/pull/4255) TorchTune validation reads the node count a TrainJob inherits from its runtime | A LoRA TrainJob on a two-node custom runtime passed admission, then ran the full multi-node fine-tune recipe; now admission rejects it | 2026-10-23 |
| [llm-d-router](https://github.com/llm-d/llm-d-router) | [#3476](https://github.com/llm-d/llm-d-router/pull/3476) `autoTune` per-pod LRU capacity is now sized in router blocks, not engine blocks | 31250 engine blocks of 16 tokens gave a 31250-entry LRU, now 7812 entries of 64 tokens; `TestMakeserver_AutotuneCapacityInRouterBlocks` failed before | 2026-10-20 |

### Infrastructure

| Project | Change | Result | Date |
|---|---|---|---|
| [oras-go](https://github.com/oras-project/oras-go) | [#1721](https://github.com/oras-project/oras-go/pull/1721) an empty `credHelpers` entry selects the config file, as in docker/cli, instead of the `credsStore` helper | With `{"credsStore":"fake","credHelpers":{"h1.example.com":""}}`, `Get`, `Put` and `Delete` for `h1.example.com` ran the `fake` helper; now they use the config file | 2026-10-25 |
| [unpoller](https://github.com/unpoller/unpoller) | [#1122](https://github.com/unpoller/unpoller/pull/1122) the InfluxDB `uap` point of a UDM no longer carries a `state` field the clients cannot encode | A UDM's built-in access point wrote `state="{1 1}"` as text; now `uap` has no `state` and the number stays on the `usg` point | 2026-10-25 |
| [DNSControl](https://github.com/DNSControl/dnscontrol) | [#5034](https://github.com/DNSControl/dnscontrol/pull/5034) `D_EXTEND()` ignores the empty tag on either name, since `example.com` and `example.com!` are documented as the same | `D("example.com!")` with `D_EXTEND("sub.example.com")` put `www` at `www.su`, and the reverse made `dnscontrol check` panic; now both give `www.sub` | 2026-10-25 |
| [Soup Sieve](https://github.com/facelessuser/soupsieve) | [#330](https://github.com/facelessuser/soupsieve/pull/330) `:lang()` stops at an element with an empty `lang` instead of taking an ancestor's language | `p:lang(de)` matched a `<p lang="">` inside `<div lang="de">`; now it does not, as in Chrome 155, Firefox 158 and Safari 26.6 | 2026-10-25 |

## Full record

The full list of merged pull requests, by area, with a write-up where there is one: [spicyshark.co/open-source](https://spicyshark.co/open-source/).
