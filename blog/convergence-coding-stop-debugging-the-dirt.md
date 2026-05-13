# Convergence Coding: Stop Debugging the Dirt

The first time I tried vibe coding, I wondered what it would look like for infrastructure. Real infrastructure: bash, Ansible, switches, LANs, NICs, clusters. I wanted to type what I meant and get working infrastructure back.

Something like:

> Create a network with one SONiC switch and two separate LANs. In one LAN, add two Ubuntu nodes. In the other, add one Ubuntu node. Install k3s on all three nodes and configure them as a single cluster. Install nginx with anti-affinity across three pods. Use DHCP. Make sure the LANs are physically separated through separate NICs. Test everything.

The appeal was obvious, especially because networking is not my strongest area. Though, the problem is not only that infrastructure is hard for me: the infrastructure coding loop is broken.

## The missing loop

For a lot of modern application development, the loop has become almost invisible. With Node.js, Spring Boot, and similar stacks, you often do not even think about “running” the thing anymore. You save a file, live reload picks it up, the app redeploys, and the result is in front of you. 

Infrastructure has improved too. Infrastructure as code made systems more repeatable, reviewable, and automatable, yet the loop is still usually measured in tens of minutes or more. Infrastructure work often has to create the world before it can test the world. You do not just run code. You provision machines, attach NICs, wire networks, boot operating systems, wait for services, configure dependencies, and only then discover whether the system matches the intent.

Then comes the dirty environment problem. After a few rounds of testing, do you rebuild everything from scratch and pay the time cost again? Or do you keep iterating on an environment that may already contain leftovers from previous attempts? Every failed attempt can leave residue: modified config, stale leases, changed routes, half-installed packages, old cluster state, broken services, orphaned files.

**Keep going, and you may be debugging the dirt instead of the system.**

That mismatch is what got me interested. Could infrastructure get the same kind of fast, clean feedback loop that software developers take for granted? [Here is what that loop looks like when it is compressed: an agent provisions the cluster, wires the network, installs k3s, checks the result, and corrects course while the task is still fresh.](https://www.youtube.com/watch?v=8nts8oI-yeA)

## Antrieb

That question turned into seven months of work. The result is Antrieb, German for drive or propulsion.

In plain terms, Antrieb gives an AI agent or LLM a fast, disposable lab where it can build and test real infrastructure. It is an MCP server that lets LLMs and agents create VM-based infrastructure: routers, switches, NICs, individual VMs, and small VM clusters. 

The two properties I found critical to focus on are fidelity and speed:

1. Fidelity: fidelity means the environment behaves enough like real infrastructure for the result to mean something. A huge part of computing systems still runs in VMs. A huge part of networking, edge systems, appliance software, programmable switches, routers, firewalls, and real-world infrastructure is still best represented in VMs.

2. Speed: speed matters because fidelity alone does not give you a usable loop. If every attempt takes too long, you stop iterating. You start trying to be right up front instead of using the loop to discover what is wrong.

Getting fidelity and speed together is the hard part. High fidelity is costly and usually means more setup, more state, more boot time, more moving parts. Speed usually comes from simplifying those things away.

Getting to subsecond launch took a while. I tried a number of techniques: kernel IP injection, Kernel Samepage Merging (KSM), microVMs, Kata Containers, Firecracker, and direct kernel boot. Snapshotting was the one that gave me the speed and scalability across the distro and version landscape. Now, a 4-node cluster launches in under 2 seconds. On the networking side, Antrieb uses OpenVSwitch, which gives the LLM the flexibility to wire up a wide variety of networks: different topologies, NIC layouts, and isolation patterns, rather than being locked into one shape.

Examples of supported base images are AlmaLinux, Ubuntu, SONiC, Debian, and Alpine. On top of those, you can build stack images: Ansible Controller, MariaDB, Vault, and so on, so heavy installs happen once and not on every run. With that loop, an agent can instantly provision a cluster, network, switches, and NICs. It can run commands, install packages, inspect the result, change its approach, and try again while the task is still fresh.

## Convergence coding

An LLM can produce plausible Ansible, plausible bash, or plausible network configuration. But plausible is not the same as correct.

**Generation produces a hypothesis.**

The LLM proposes a system: these machines, these NICs, these networks, these commands, this cluster state. That hypothesis might be right. It might be subtly wrong. It might satisfy the prompt while missing the intent.

**Convergence coding is the loop around that hypothesis: conformance checking, completeness checking, and refinement.**

### Conformance checking

Conformance checking asks whether the LLM did what you asked. Instead of reviewing generated code and guessing whether it creates the right system, you inspect the actual infrastructure. Are the nodes connected correctly? Did DHCP work? Is k3s actually running? Are the nginx pods scheduled with anti-affinity?

For example, the agent may claim it created two separate LANs. Conformance checking means inspecting the actual interfaces, bridges, routes, and leases to see whether that is true.

### Completeness checking

Completeness checking asks whether you asked for what you actually wanted. Sometimes the generated system satisfies the prompt, and that is how you discover the prompt was wrong. The problem is not always that the LLM missed the request. Sometimes the request missed your intent.

For example, you inspect the cluster and realize the LLM deployed one control plane and two workers. What you actually wanted was for the control plane node to also run as a worker, so you have three workers for your three pods. The prompt was satisfied. The intent was not.

### Refinement

Refinement is how larger systems become manageable. You give the LLM a smaller piece, get it right, save it as a runbook, and move on.

First the cluster: one SONiC switch, two LANs, three Ubuntu nodes wired through separate NICs. Then k3s: installed on all three nodes, control plane also running as a worker. Then the application: nginx with anti-affinity across three pods, plus an ingress so the service is reachable from outside.

Each stage is checked and saved before the next begins. When all three are verified, the LLM can run them end to end using the saved runbooks. For a full run, including the agent’s commands, checks, mistakes, and corrections, I saved the Claude Code execution trace here:

[Full SONiC dual LAN network with k3s and nginx](https://github.com/jade-pico/antrieb/blob/main/runs/sonic-dual-lan-k3s-nginx.md)

Convergence coding is different from vibe coding. Vibe coding can wander. Infrastructure has to converge. Antrieb exists to make that convergence loop fast enough that you actually use it.

## Try it

To try it, add the MCP connector in Claude.ai as using this MCP url https://antrieb.sh/mcp. Follow the instructions at: https://www.youtube.com/watch?v=8nts8oI-yeA


For Claude Code or Codex, sign in at https://antrieb.sh/dash, get an API key, connect the MCP, and try the SONiC/k3s prompt above.

I am curious what people build with it, what breaks, and where the loop still feels too slow.
