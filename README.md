### spicysharkco

Infrastructure and performance for AI systems: Kubernetes, LLM inference, coding agents, robotics.
Go · Rust · Python · C++ · TypeScript · Write-ups at [spicysharkco.github.io](https://spicysharkco.github.io)

#### Infrastructure

| Project | Merged change | Result | Story |
|---|---|---|---|
| [kube-router](https://github.com/cloudnativelabs/kube-router) | [#2192](https://github.com/cloudnativelabs/kube-router/pull/2192) network-policy sync stops copying the iptables table once per pod | 320x faster, 99.8% less allocation (benchmark) | [post](https://spicysharkco.github.io/2026/10/15/kube-router-netpol-sync-copy.html) |
| [k3s](https://github.com/k3s-io/k3s) | [#14801](https://github.com/k3s-io/k3s/pull/14801) `--disable-etcd` on an existing server removes its etcd member | no stale member left in the cluster | |
| [k3s](https://github.com/k3s-io/k3s) | [#14811](https://github.com/k3s-io/k3s/pull/14811) tunnel proxy stops leaking a goroutine when both directions of a connection fail | 1,029 leaked goroutines → 0 (10,000-run test) | |
| [redis-py](https://github.com/redis/redis-py) | [#4434](https://github.com/redis/redis-py/pull/4434) asyncio client breaks the reference cycle on each error reply | 1,000 leaked exceptions → 0 (GC off) | [post](https://spicysharkco.github.io/2026/10/15/redis-py-asyncio-error-reference-cycle.html) |

#### AI coding agents

| Project | Merged change |
|---|---|
| [oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex) | [#3791](https://github.com/Yeachan-Heo/oh-my-codex/pull/3791) HUD caches the git branch label until HEAD, refs or config change |
| [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) | [#4373](https://github.com/Yeachan-Heo/oh-my-claudecode/pull/4373) hooks start file-context injectors only for the tools they act on |
| [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) | [#10133](https://github.com/code-yeongyu/oh-my-openagent/pull/10133) git diff stats read NUL-separated, so renamed and quoted paths keep their counts |

#### Robotics

| Project | What | Result | Story |
|---|---|---|---|
| [slam_toolbox](https://github.com/SteveMacenski/slam_toolbox) (ROS 2) | [#905](https://github.com/SteveMacenski/slam_toolbox/pull/905) node removal stops copying the pose graph's vertex map and edge list | 84x faster per removal on a 19,870-vertex map (benchmark) | |
| [chess-sim2real-benchmark](https://github.com/spicysharkco/chess-sim2real-benchmark) · [dataset](https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark) | render generator and 20,000-image sim-to-real benchmark | a colour-convention fix took per-square accuracy from 54.4 to 85.2 | [post](https://spicysharkco.github.io/2026/10/15/colour-convention-looked-like-domain-gap.html) |

#### Tools

- [sharkline](https://github.com/spicysharkco/sharkline): Claude Code statusline in Rust.
- [sharkswap](https://github.com/spicysharkco/sharkswap): Claude Code account switcher.
