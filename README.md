### spicysharkco

Infrastructure and performance engineer for AI systems: Kubernetes networking and control planes, LLM inference,
AI coding-agent tooling, and sim-to-real perception for robotics. Go · Rust · Python · TypeScript.

#### Kubernetes

- **[kube-router](https://github.com/cloudnativelabs/kube-router)** [#2192](https://github.com/cloudnativelabs/kube-router/pull/2192):
  removed a per-pod copy of the iptables filter table from network-policy sync: ~300x faster, 99.8% less memory.
- **[k3s](https://github.com/k3s-io/k3s)** [#14801](https://github.com/k3s-io/k3s/pull/14801):
  `--disable-etcd` on an existing server now removes its etcd member instead of leaving a stale one in the cluster.

#### AI coding agents

Merged fixes and performance work in the multi-agent orchestration layers
[oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode), [oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex)
and [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent), including:

- **oh-my-codex** [#3791](https://github.com/Yeachan-Heo/oh-my-codex/pull/3791):
  HUD caches the git branch label until HEAD, refs or config change.
- **oh-my-claudecode** [#4373](https://github.com/Yeachan-Heo/oh-my-claudecode/pull/4373):
  hooks start file-context injectors only for the tools they act on.
- **oh-my-openagent** [#10133](https://github.com/code-yeongyu/oh-my-openagent/pull/10133):
  git diff stats read NUL-separated, so renamed and quoted paths keep their counts.

#### Robotics perception

- **[chess-sim2real-benchmark](https://github.com/spicysharkco/chess-sim2real-benchmark)**
  ([dataset](https://huggingface.co/datasets/spicysharkco/chess-sim2real-benchmark)):
  a render generator and a 20,000-image benchmark that measures what synthetic training data buys and costs
  when a perception model moves from renders to a camera. One finding: a board colour-convention mismatch
  looked like a domain shift; fixing one renderer expression raised per-square accuracy from 54.4 to 85.2.

#### Tools

- **[sharkline](https://github.com/spicysharkco/sharkline)**: Claude Code statusline in Rust.
- **[sharkswap](https://github.com/spicysharkco/sharkswap)**: Claude Code account switcher.
