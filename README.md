### spicysharkco

I build products and fix the infrastructure under them upstream: Kubernetes, LLM inference, coding agents, robotics.
Go · Rust · Python · C++ · TypeScript · Write-ups at [spicysharkco.github.io](https://spicysharkco.github.io)

#### Infrastructure

| Project | Merged change | Result | Story |
|---|---|---|---|
| [kube-router](https://github.com/cloudnativelabs/kube-router) | [#2192](https://github.com/cloudnativelabs/kube-router/pull/2192) network-policy sync stops copying the iptables table once per pod | 320x faster, 99.8% less allocation (benchmark) | [post](https://spicysharkco.github.io/2026/10/15/kube-router-netpol-sync-copy.html) |
| [k3s](https://github.com/k3s-io/k3s) | [#14801](https://github.com/k3s-io/k3s/pull/14801) `--disable-etcd` on an existing server removes its etcd member | no stale member left in the cluster | |
| [k3s](https://github.com/k3s-io/k3s) | [#14811](https://github.com/k3s-io/k3s/pull/14811) tunnel proxy stops leaking a goroutine when both directions of a connection fail | 1,029 leaked goroutines → 0 (10,000-run test) | [post](https://spicysharkco.github.io/2026/10/16/k3s-proxy-goroutine-leak.html) |
| [kine](https://github.com/k3s-io/kine) | [#800](https://github.com/k3s-io/kine/pull/800) compaction retries on MariaDB error 1020, so kine runs on MariaDB 11.8 | 44 failed compactions → 0 in 45 s of concurrent writes | |
| [cri-tools](https://github.com/kubernetes-sigs/cri-tools) (crictl) | [#2212](https://github.com/kubernetes-sigs/cri-tools/pull/2212) `make verify` checks docs and vendor drift in a temp dir instead of rewriting the tree | tree byte-identical after every run | |
| [redis-py](https://github.com/redis/redis-py) | [#4434](https://github.com/redis/redis-py/pull/4434) asyncio client breaks the reference cycle on each error reply | 1,000 leaked exceptions → 0 (GC off) | [post](https://spicysharkco.github.io/2026/10/15/redis-py-asyncio-error-reference-cycle.html) |
| [sql_exporter](https://github.com/burningalchemist/sql_exporter) | [#1115](https://github.com/burningalchemist/sql_exporter/pull/1115) log context trims its leading comma again, so failed queries stop logging a false "Invalid context log item" warning | 1 false warning per failed query → 0 (test) | |

#### AI coding agents

| Project | Merged change |
|---|---|
| [oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex) | [#3791](https://github.com/Yeachan-Heo/oh-my-codex/pull/3791) HUD caches the git branch label until HEAD, refs or config change · [all 4 merged](https://github.com/Yeachan-Heo/oh-my-codex/pulls?q=is%3Apr+author%3Aspicysharkco+is%3Amerged) |
| [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) | [#4373](https://github.com/Yeachan-Heo/oh-my-claudecode/pull/4373) hooks start file-context injectors only for the tools they act on · [all 7 merged](https://github.com/Yeachan-Heo/oh-my-claudecode/pulls?q=is%3Apr+author%3Aspicysharkco+is%3Amerged) |
| [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) | [#10133](https://github.com/code-yeongyu/oh-my-openagent/pull/10133) git diff stats read NUL-separated, so renamed and quoted paths keep their counts · [all 2 merged](https://github.com/code-yeongyu/oh-my-openagent/pulls?q=is%3Apr+author%3Aspicysharkco+is%3Amerged) |

#### Robotics

| Project | What | Result | Story |
|---|---|---|---|
| [OpenCV](https://github.com/opencv/opencv) (calib3d) | [#30316](https://github.com/opencv/opencv/pull/30316) `validateDisparity` skips out-of-range disparities instead of indexing past the row buffers | fixes a crash reported in April 2024 ([#25367](https://github.com/opencv/opencv/issues/25367)); two earlier fixes were closed | [post](https://spicysharkco.github.io/2026/10/16/opencv-validatedisparity-out-of-bounds.html) |
| [slam_toolbox](https://github.com/SteveMacenski/slam_toolbox) (ROS 2) | [#905](https://github.com/SteveMacenski/slam_toolbox/pull/905) node removal stops copying the pose graph's vertex map and edge list | 84x faster per removal on a 19,870-vertex map (benchmark) | |
| [chess-sim2real-benchmark](https://github.com/spicysharkco/chess-sim2real-benchmark) · [dataset](https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark) | render generator and 20,000-image sim-to-real benchmark | a colour-convention fix took per-square accuracy from 54.4 to 85.2 | [post](https://spicysharkco.github.io/2026/10/15/colour-convention-looked-like-domain-gap.html) |

#### Tools

- [sharkline](https://github.com/spicysharkco/sharkline): Claude Code statusline in Rust.
- [sharkswap](https://github.com/spicysharkco/sharkswap): Claude Code account switcher.
