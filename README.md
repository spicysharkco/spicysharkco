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
  <a href="#selected-fixes">Selected fixes</a> · <a href="#my-own-projects">My projects</a> · <a href="#research-in-progress">Research in progress</a> · <a href="#upstream-work">Upstream work</a>
</p>

## Selected fixes

| Project | Bug | Result | Merged |
|---|---|---|---|
| [kube-router](https://github.com/cloudnativelabs/kube-router/pull/2192) | Network-policy sync copied and scanned the whole iptables-save buffer once per local pod | About 320x faster and 99.8% less allocation per op (benchmark: 80 pods, 1 MiB of existing rules) | 2026-10-12 |
| [k3s](https://github.com/k3s-io/k3s/pull/14811) | Goroutine leak and data race in the server's tunnel proxy | On multi-core runners the old code left 1029 goroutines behind in the new test; the fix passes `go test -race -count=20` 20 of 20 | 2026-10-15 |
| [OpenCV](https://github.com/opencv/opencv/pull/30316) | Out-of-bounds access in `validateDisparity`, the stereo-depth left-right check | AddressSanitizer: all 4 new test cases overflow without the fix, all 4 pass with it | 2026-10-16 |
| [llm-d-router](https://github.com/llm-d/llm-d-router/pull/3476) | Prefix-cache index sized in engine blocks instead of router blocks | With vLLM's default block size the index could track 4x the tokens the engine's KV cache held (up to 64x at block size 1); 3 of 6 new test cases failed on main, all 6 pass with the fix | 2026-10-20 |

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
- [PR #3476](https://github.com/llm-d/llm-d-router/pull/3476), merged 2026-10-20

</details>

## My own projects

- [sharkline](https://github.com/spicysharkco/sharkline): a status line for Claude Code, in Rust. A warm render takes about 1.3 ms with no child process and no network; a Node process that does nothing takes about 30 ms on the same machine. Try: `cargo install --locked --git https://github.com/spicysharkco/sharkline`.
- [sharkswap](https://github.com/spicysharkco/sharkswap): a Claude Code account switcher in Rust. It swaps between saved accounts, shows each account's 5-hour and weekly usage, and can move the login when one runs out. One binary, no daemon. Try: `cargo install --locked --git https://github.com/spicysharkco/sharkswap`, then `sharkswap probe`.
- [chess-sim2real-benchmark](https://github.com/spicysharkco/chess-sim2real-benchmark): a render generator and a 20,000-image benchmark for chess-position recognition, with [a paper on what synthetic training data buys and costs](https://github.com/spicysharkco/chess-sim2real-benchmark/blob/main/paper/main.pdf) ([dataset](https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark) <a href="https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark"><img alt="Hugging Face downloads" src="https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fhuggingface.co%2Fapi%2Fdatasets%2Fspicysharkco%2Fchess-sim2real-benchmark%3Fexpand%255B%255D%3DdownloadsAllTime&query=%24.downloadsAllTime&label=Hugging%20Face%20downloads&color=0a1626&style=flat-square"></a>). The README walks through rendering and scoring four frozen views locally.

## Research in progress

- **Quantization.** GGUF files that give each tensor group its own quant type, chosen by measured sensitivity, inside a fixed file size ([model card](https://huggingface.co/spicysharkco/Qwen3.8-27B-OBLITERATED-DYN-GGUF)). At the same size as the upstream Q4_K_M (16.81 GB), mean KL divergence from BF16 fell from 0.0348 to 0.0195 on 30,690 held-out tokens. About two thirds of that gain comes from the importance matrix; the rest comes from the per-tensor mix. CPU prompt processing runs at 0.85x the upstream file's speed. Limit: the evaluation is text only, and no task benchmark was run.
- **Chess benchmark.** Changing which corner is light in the renderer moved per-square accuracy from 54.4% to 85.2%, and only 18 of 192 boards are exact after the fix. Limit: the public reproduction covers a bounded render and scoring path, not all 20,000 cells or the adaptation-training study.

## Upstream work

Merged fixes, grouped by area. Each name links to its merged pull requests.

- **Kubernetes and cloud native:** [k3s](https://github.com/k3s-io/k3s/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [kube-router](https://github.com/cloudnativelabs/kube-router/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Cluster API](https://github.com/kubernetes-sigs/cluster-api/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [external-dns](https://github.com/kubernetes-sigs/external-dns/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [cert-manager](https://github.com/cert-manager/cert-manager/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Linkerd](https://github.com/linkerd/linkerd2/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Dapr](https://github.com/dapr/dapr/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco)
- **AI and ML:** [AIBrix (vLLM project)](https://github.com/vllm-project/aibrix/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [llm-d Router](https://github.com/llm-d/llm-d-router/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Kueue](https://github.com/kubernetes-sigs/kueue/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [KubeAI](https://github.com/kubeai-project/kubeai/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Kubeflow Trainer](https://github.com/kubeflow/trainer/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Hugging Face Text Embeddings Inference](https://github.com/huggingface/text-embeddings-inference/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [ONNX](https://github.com/onnx/onnx/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [OpenCV](https://github.com/opencv/opencv/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco)
- **Clients and observability:** [redis-py](https://github.com/redis/redis-py/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [go-redis](https://github.com/redis/go-redis/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco), [Prometheus Go client](https://github.com/prometheus/client_golang/pulls?q=is%3Apr+is%3Amerged+author%3Aspicysharkco)

The full list, with write-ups: [spicyshark.co/open-source](https://spicyshark.co/open-source/).
