# Vultr B200: What You Actually Get, What It Costs Per Hour, and Who Should Skip the 8-GPU Node (192 GB vs 179 GB Explained)

Vultr sells the B200 as one thing: a complete NVIDIA HGX B200 node with eight Blackwell GPUs, billed by the hour, deployed self-service with no reservation and no sales call. There is no single-GPU option on that page, so the real question is whether your workload can use a full 8-GPU NVSwitch system.

If it can, the setup is unusually frictionless for this class of hardware. If it can't, you'll be paying for seven idle GPUs. Here's what the node contains, how billing works, which spec numbers on the marketing page mean something different once you're inside the machine, and where the practical limits sit.

> **Quick version:** Vultr offers the B200 in two forms, Vultr Cloud GPU and Vultr Bare Metal, both as 8-GPU nodes with 1.8 TB/s per-GPU NVLink, hourly billing and self-service provisioning. It suits teams already committed to multi-GPU training or large-model inference. Anyone running a single 7B–30B model should look at a smaller GPU instead.
> 👉 [Check the Vultr B200 node page and the current hourly rate](https://www.vultr.com/products/cloud-gpu/nvidia-b200/?ref=9738262-9J)

## The node, in one table

Vultr doesn't tier the B200 into small/medium/large plans. It's one hardware specification sold under two deployment models.

| Deployment form | GPUs | Memory per GPU | GPU interconnect | Price and billing | Start |
| --- | --- | --- | --- | --- | --- |
| Vultr Cloud GPU (HGX B200) | 8× NVIDIA Blackwell | Up to 192 GB HBM3e (~179 GiB usable) | 1.8 TB/s per GPU, 14.4 TB/s aggregate, NVSwitch 5.0 | Hourly, per node; live rate at checkout | [Deploy a Cloud GPU B200 node](https://www.vultr.com/products/cloud-gpu/nvidia-b200/?ref=9738262-9J) |
| Vultr Bare Metal (HGX B200) | 8× NVIDIA Blackwell | Up to 192 GB HBM3e (~179 GiB usable) | 1.8 TB/s per GPU, 14.4 TB/s aggregate, NVSwitch 5.0 | Hourly, per node; live rate at checkout | [Provision a Bare Metal B200 node](https://www.vultr.com/products/cloud-gpu/nvidia-b200/?ref=9738262-9J) |

The same page also confirms 2×7 NVDEC and 2×7 NVJPEG decoders per GPU, FP8 Tensor Core at 72 petaFLOPS and INT8 at 72 petaOPS (both with sparsity), and a note that the specifications are preliminary and subject to change.

## Hourly billing, and why you won't see a number here

Vultr's B200 product page shows a starting hourly rate, but the figure is formatted in a way that can't be resolved to a single value from the public page, and there is no second published price point to cross-check against. Quoting a wrong number would be worse than quoting none, so treat the console or checkout screen as the authoritative rate.

What is confirmed:

- Billing is **hourly**, and the unit is the **node**, not the GPU.
- Provisioning is **self-service**. Vultr states plainly that no reservations or special requests are required to deploy.
- Both **Cloud GPU and Bare Metal** forms are available through Vultr Clusters.
- No monthly or annual commitment, minimum term, or reserved-instance discount appears on the product page.

That last point matters for budgeting. Hourly on-demand pricing is the flexible option and almost never the cheapest per GPU-hour if you plan to run the node flat out for months. Nothing published suggests Vultr currently offers a committed-use discount on B200, so plan your cost model on on-demand hours times expected runtime, then confirm at checkout.

## 192 GB or 179 GB? Reading the B200 spec sheet properly

This is where most write-ups recycle the headline numbers without explaining them. Four figures on Vultr's own materials need translation before you size a model against them.

**192 GB vs 179 GB of VRAM.** The marketing page says "up to 192 GB HBM3e per GPU." Vultr's hardware documentation for the same GPU reports 179 GB HBM3e, and the verified instance shows 183,359 MiB per GPU. The gap is decimal GB versus binary GiB: 183,359 MiB works out to roughly 179 GiB, and the physical HBM3e stack is a 192 GB part. Budget your model weights, KV cache and activations against ~179 GiB per GPU, or ~1.43 TB for the whole node. A 192 GB assumption will bite you at the worst possible moment.

**144 petaFLOPS of FP4.** That is the aggregate figure for eight GPUs **with sparsity** enabled, and Vultr footnotes it as such. The per-GPU dense NVFP4 number in Vultr's docs is about 9.0 PFLOPS, which puts eight cards near 72 PFLOPS dense. The 144 figure isn't wrong, it's a structured-sparsity peak. If your kernels don't exploit 2:4 sparsity, plan on the lower number.

**1.5 TB and 64 TB/s vs 1.4 TB and 60 TB/s.** The spec table says up to 1.5 TB of fast memory and up to 64 TB/s aggregate bandwidth. The key-benefits section of the same datasheet, and the discover page, say 1.4 TB combined memory and 60 TB/s. The measured configuration reads ~1.43 TB total, and eight GPUs at 8.0 TB/s each arithmetically gives 64 TB/s. Treat 1.4 TB / 60 TB/s as the planning number and the higher pair as the spec-sheet ceiling.

**34 data center regions.** The current product page lists 34 regions across six continents. The March 2026 datasheet says 33, and the March 2025 launch announcement said 32. The count has been growing; 34 is what the page shows now.

## Constraints you only find in the documentation

Vultr's hardware overview documents the node as it actually behaves, and a few of these details change how you'd deploy.

**Power draw is real.** Each GPU pulls up to 1,000 W under load and around 140 W at idle. For the node that's roughly 1.12 kW idle, 1.6 kW under light load, and 5.6–8.0 kW at full load. Vultr handles the cooling, but the numbers matter if you're comparing token-per-watt efficiency across providers or estimating what a dense cluster deployment implies.

**The node is split across two NUMA domains.** GPUs 0–3 sit on NUMA node 0 (CPUs 0-63 and 128-191), GPUs 4–7 on NUMA node 1 (CPUs 64-127 and 192-255). Pin your processes accordingly. If you're running disaggregated prefill/decode serving, splitting the two pools across the two halves keeps CPU-side work local to the GPUs doing it.

**Everything inside the node is NVSwitch.** Every GPU pair reports `NV18`, meaning 18 bonded NVLinks through NVSwitch 5.0, giving 1.8 TB/s bidirectional between any two GPUs. That's what makes tensor parallelism practical. Cross-node traffic is a different path: Vultr Clusters connect nodes over NVIDIA InfiniBand, and the machine carries 14 Mellanox ConnectX NICs (mlx5_0 through mlx5_13) for it.

**The software baseline is recent.** The verified instance runs driver 580.105.08, CUDA 13.0, persistence mode enabled, compute capability sm_100 (10.0). Containers built against sm_90 need rebuilding. Vultr's inference cookbook recommends vLLM 0.16.0 for this GPU and documents a PyTorch 2.9.1+cu130 stack.

## What the performance claims actually say

Vultr and NVIDIA publish four comparison figures, all measured against Hopper-generation hardware, none of them independently benchmarked:

| Workload | Claim | Comparison |
| --- | --- | --- |
| GPT-MoE-1.8T training | 3x faster | vs NVIDIA Hopper GPUs |
| GPT-MoE-1.8T inference | 15x faster | vs NVIDIA Hopper GPUs |
| Inference energy use | up to 12x less | at similar performance vs Hopper |
| HPC and data analytics | 2x faster | vs NVIDIA Hopper GPUs |

The query benchmark claim is separate: 6x faster than CPUs and 2x faster than NVIDIA H100, credited to Blackwell's dedicated Decompression Engine and its support for current compression formats.

The 3x training figure is the one I'd trust most for planning, because training throughput on a fixed model is the easiest thing to measure consistently and the least sensitive to workload shape. The 15x inference number depends heavily on what "real-time inference" means in that test, and the 12x energy saving is explicitly conditioned on "similar performance." Neither is a promise about your model.

One architectural point is worth more than any of these ratios. LLM decode is memory-bandwidth bound, and 8.0 TB/s per GPU is the number that turns into tokens per second. Vultr's docs also note that native FP4 halves memory versus FP8, which is enough to move a model that needs two or more GPUs at FP8 onto a single GPU. That's a deployment-shape change, not just a speed claim, and it's the most useful thing FP4 does for smaller workloads.

## Getting it running

Provisioning happens through the Vultr console or API. GPU Enabled Images ship with the NVIDIA CUDA Toolkit and drivers preinstalled, so a working environment is one click from the console rather than an afternoon of driver archaeology.

For multi-node work, Vultr Clusters lets you create, modify and scale B200 clusters on demand over InfiniBand, attach high-performance storage, monitor cluster metrics, and schedule workloads with Slurm or Kubernetes. Vultr Kubernetes Engine integration covers the container-native path. Compliance coverage includes ISO and SOC 2+ frameworks.

👉 [Start with a GPU Enabled Image on a B200 node](https://www.vultr.com/products/cloud-gpu/nvidia-b200/?ref=9738262-9J)

## Who should rent this, and who shouldn't

**Reasonable fits:** trillion-parameter or otherwise large-model training where you're already sharding across eight GPUs; long-context agentic inference with multi-step planning and tool calls, which is the workload Vultr explicitly targets; HPC and analytical query workloads that benefit from the Decompression Engine; anything where a single node's 1.43 TB of pooled memory is the constraint you're designing around.

**Poor fits:** single-model inference under ~70B parameters at FP8, where one GPU is enough and seven would idle. Small fine-tuning jobs. Anything still in the prototyping phase where you'd rather iterate cheaply on a smaller card and move to B200 for the final run. The B200 node is a commitment to parallelism, and workloads that can't use eight GPUs don't get faster by having them.

## FAQ

**Can I rent a single B200 GPU from Vultr?**
Not from the B200 product. It's sold as an 8-GPU HGX node. Vultr's other GPU offerings cover smaller configurations.

**Is it billed hourly or monthly?**
Hourly, per node. No monthly or annual plan is published for the B200.

**Do I need to reserve capacity or talk to sales?**
No. Vultr states that no reservations or special requests are required, and you can provision through the console or API.

**Slurm or Kubernetes?**
Both. Vultr Clusters supports either scheduler, and Vultr Kubernetes Engine covers the managed Kubernetes route.

**How much memory does FP4 actually save?**
Half compared with FP8. That's the difference between needing two GPUs and needing one for some models.

**How much usable VRAM does the node have?**
About 179 GiB per GPU and roughly 1.43 TB across all eight, despite the 192 GB per-GPU figure on the spec sheet.

## Before you click deploy

Match the tool to the workload. If you're training or serving something that genuinely spans eight GPUs, the Vultr B200 node gives you the full NVSwitch fabric, self-service provisioning and a documented software baseline without a procurement cycle. If your model fits on one card, spend a fraction of the money elsewhere and come back when it doesn't.

Two things to confirm on the order screen, because they aren't settled on the public pages: the exact current hourly rate for the node, and whether any promotional signup credit on the account-creation path can be applied to GPU instances. Both are visible before you commit.

👉 [Create a Vultr account and check B200 availability in your region](https://www.vultr.com/register/?ref=9738262-9J)
