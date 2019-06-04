# Poor performance after containerization

## Key points

1. On the aspect of performance containerization != virtualization and demonstrates
2. "Isolation" processes compete not only for CPU, MEM, Disk and Network but also for all kinds of kernel resources.
3. All these collisions in kernel resource among lots of processes are excellent opportunities for intruders to break though "lightweight" isolation and create all kinds of convert channels between containers.
4. In these cases, we can use perf to analyze the bottlenecks in kernel space.
5. The container is highly dependent on kernel. Some of these dependences are reflected in system config (e.g. tcp session, ulimit, and so on).

## Precaustion

1. Make preventions for prevent from competing with the kernel resource: monitor the kernel stack, use vm rather than container, use native container(something like kata)

## Use perf to analyze time comsuming in stack

```bash
git clone https://github.com/brendangregg/FlameGraph.git
perf record -F 99 -p ${1} -g -- sleep 30
perf script | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > ${1}.svg
```

## Reference

- [Hackernoon -- Another reason why your Docker containers may be slow](https://hackernoon.com/another-reason-why-your-docker-containers-may-be-slow-d37207dec27f)
- [Container isolation gone wrong](https://sysdig.com/blog/container-isolation-gone-wrong/)
