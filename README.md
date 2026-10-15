### spicysharkco

AI infra · robotics · optimization

Contributor: [k3s](https://github.com/k3s-io/k3s) · [kube-router](https://github.com/cloudnativelabs/kube-router) · [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) · [oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex)

#### Projects

- **[sharkline](https://github.com/spicysharkco/sharkline)**:
  fast Claude Code statusline in Rust.
- **[sharkswap](https://github.com/spicysharkco/sharkswap)**:
  Claude Code account switcher.

#### Highlights

- **kube-router** [#2192](https://github.com/cloudnativelabs/kube-router/pull/2192):
  network-policy sync step ~300x faster, 99.8% less memory.
- **k3s** [#14801](https://github.com/k3s-io/k3s/pull/14801):
  `--disable-etcd` on an existing server now removes its etcd member.

#### How I open a pull request

- **Reproduce on the latest release first.** A bug that is already fixed is not a contribution.
- **Search open and closed issues before filing.** Duplicates cost the maintainer, not me.
- **One purpose per PR.** A reviewer should be able to hold the whole change in their head.
- **Evidence beyond green CI.** Logs, before and after numbers, and a test that fails without the fix and passes with it.
- **Hermetic tests.** No real home directory, no shared sockets, no timing bound that only holds on a fast laptop.
