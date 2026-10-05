### spicysharkco

Building more efficient AI infrastructure.

I'm an engineer working across AI, data centers and robotics. I measure where systems waste compute, fix the root cause, and send the fix upstream so everyone gets it.

#### Selected open-source work

- **kube-router** [#2192](https://github.com/cloudnativelabs/kube-router/pull/2192): removed a per-pod copy of the network-policy rules buffer. In benchmarks, that step of each policy sync runs about 300x faster and allocates 99.8% less memory.
