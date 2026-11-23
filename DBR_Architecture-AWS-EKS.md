# Digital Board Replacement (DBR) for Virtual Perimeter Advertising — Current On-Prem GPU/Kubernetes Architecture → AWS/EKS Target Architecture

**Document type:** reference architecture + risk register + migration design

**Scope:** live football, near-real-time digital board replacement (DBR), per-territory ad feeds, GPU video pipeline, Kubernetes on both sides.

---

## 0. How to read this document (and what I am not asserting)

The markers distinguish facts, assumptions, recommendations and items that must be verified:

| Marker | Meaning |
| --- | --- |
| (no marker) | Engineering context, standard practice, or arithmetic derived from stated inputs; not confirmation of your actual environment |
| [A] | Business or workload assumption used to make the design concrete. Change it and the numbers may change. |
| [R] | Recommendation or design judgement; validate it against measured workload behaviour and contractual requirements. |
| [V] | Verify at design freeze. Product capabilities, quotas, instance availability, support matrices and prices change. |

### 0.1 Assumptions I am making explicit [A]

1. **Software/CV-based replacement on a received video feed** is the workload model used here. Hardware-assisted approaches (for example, IR markers on boards and a compatible camera/OB chain) have venue- and production-specific dependencies; do not assume they can be moved to a remote cloud pipeline or reproduced from a standard PGM feed. Confirm whether such a system is used and where its keying/compositing stage sits in the signal chain (see §6).
2. You receive the **host broadcaster PGM ("world") feed**, dirty with physical ads, plus audio and ancillary data. You may receive some ISO camera feeds. [A]
3. **Delivery format:** 1080i50 and/or 1080p50, 10-bit 4:2:2, BT.709; some UHD/HLG events. European football contribution is commonly 1080i50; this matters enormously (§1.4).
4. **K = 5–25 territory outputs per match, N = 1–30 concurrent matches** at peak.
5. **"Near real time" = sub-second glass-to-glass added latency**, frame-continuous, no visible discontinuity. I design for ≤ ~500 ms added, and show what changes if you need ≤ 100 ms.
6. Current Kubernetes is on **bare metal / self-managed VMs** in your own facility, with self-managed control plane, CNI, CSI, GPU stack.
7. Ads are **pre-sold rotations** (not per-impression programmatic decisioning in the live path).

### 0.2 Facts that constrain any architecture (these are not negotiable)

- **Speed of light in fibre** ≈ 200,000 km/s ⇒ ~5 µs/km, ~1 ms per 200 km one way. No cloud region choice removes propagation delay.
- **Geostationary satellite** one-way ≈ 240 ms (2 × 35,786 km / c) plus modem/FEC/interleaver delay. If contribution is satellite, that dominates your entire latency budget before you process anything.
- **A 50 fps progressive stream has a 20 ms frame interval; 1080i50 has 50 fields/s, also 20 ms apart, paired into 25 interlaced frames/s.** A processing stall longer than one interval causes a missed deadline unless a jitter buffer absorbs it by adding latency.
- **AWS does provide limited multicast through Transit Gateway**, but this is not a broadcast-grade ST 2110 fabric: the documented multicast limits include a non-adjustable 1 Gb/s per flow and 20 Gb/s aggregate per AZ (plus per-flow packet-rate limits), multicast is not routed over Direct Connect attachments, and fragmented multicast packets are dropped. A single uncompressed 1080i50/1080p50 ST 2110-20 flow (~1.1–2.5 Gb/s, §1.3) therefore already exceeds the per-flow limit. AWS VPC networking also does not provide SDI, genlock or a general-purpose SMPTE ST 2059-2 timing plant. Amazon Time Sync Service provides NTP; supported Linux EC2 instances in a `precision-time` placement group can expose a PTP Hardware Clock (PHC) with microsecond-class synchronization. That is not equivalent to a broadcast grandmaster/genlock chain. [V: validate the exact instance family, AZ and placement-group support.]
- **Do not assume live migration for GPU-backed EC2 instances.** Design for scheduled maintenance, stop/start or replacement events, and retain an independent hot path; confirm lifecycle behaviour for the exact instance family.
- **NVENC/NVDEC are fixed-function engines separate from CUDA cores.** They only accelerate supported codecs, chroma formats, bit depths and profiles. The actual bottleneck may be codec engines, CUDA inference, compositing, memory traffic, host decode/encode or network I/O; measure the full pipeline.
- **MIG is GPU-specific, not universal.** A10G/L4/L40S do not provide MIG, while NVIDIA lists Ampere A30/A100, Hopper and selected Blackwell GPUs—including RTX PRO 4500/6000 Blackwell—as MIG-capable. The RTX PRO 4500 Blackwell supports up to two MIG instances and RTX PRO 6000 Blackwell up to four. [V: verify MIG mode, driver, GPU profile and EKS DRA support on the exact EC2 SKU.] DRA is a Kubernetes allocation API; it does not itself create hardware isolation.

---

### 0.3 Review notes and current platform validation (reviewed 2026-11-20)

This is a reference architecture, not an as-built audit. Section 2 is an illustrative on-prem pattern based on the assumptions in §0.1; confirm it against the actual network, GPU, codec, timing and operational inventory before using it as a migration baseline.

The following changes are material to the target design:

- **AWS GPU shortlist has moved on.** EC2 G7 uses NVIDIA RTX PRO 4500 Blackwell Server Edition GPUs and G7e uses RTX PRO 6000 Blackwell Server Edition GPUs. AWS announced G7 availability in Europe (London) on 2026-11-10 and G7e availability in London on 2026-05-07. Regional availability does not prove capacity in a particular AZ or instance size. AWS lists both G7 and G7e in Europe (London, Spain, Frankfurt, Stockholm) in the announcements checked, with up to 8 GPUs per instance (32 GB per GPU on G7, 96 GB per GPU on G7e); per-size GPU counts still need to be checked against the instance specification page [V].
- **4:2:2 must be treated as a hard codec gate.** NVIDIA's current Video Encode/Decode Support Matrix lists H.264/HEVC 4:2:2 encode and decode for the Blackwell RTX PRO 4500/6000 server GPUs, but not for A10, L4 or L40S NVENC encode. If the contribution or output is 10-bit 4:2:2, benchmark the exact decoder/encoder path before sizing.
- **MediaConnect protocol details matter.** Current documentation lists RIST Simple profile only (and no RIST encryption), supports SRT and Zixi variants, and supports ST 2110 JPEG XS flows. That is not support for generic uncompressed ST 2110-20. MediaConnect Merge requires binary-identical streams; documented failover detection is 500 ms without source data, so neither should be assumed to meet a one-frame DBR A/B switch target. Source failover is also protocol-dependent: SRT sources support failover mode only (no merge), and ST 2110 JPEG XS and CDI sources accept a single source with no source failover.
- **Capacity products are not interchangeable.** ODCRs reserve capacity for a specified AZ and instance configuration. Capacity Blocks for ML apply only to their published instance/Region list; do not assume G-family eligibility. Savings Plans are billing discounts, not capacity reservations.
- **AWS timing and multicast are limited but not absent.** Precision-time placement groups can expose PHC on supported Linux instances. Transit Gateway supports multicast for defined VPC subnet use cases, but quotas and routing limitations make it unsuitable as a general ST 2110 plant without a specific proof of fit.

External references checked for this review (re-check at design freeze):

- [AWS Elemental MediaConnect protocols](https://docs.aws.amazon.com/mediaconnect/latest/ug/protocols.html)
- [AWS Elemental MediaConnect source failover and merge semantics](https://docs.aws.amazon.com/mediaconnect/latest/ug/source-failover.html)
- [Amazon Time Sync Service and PTP Hardware Clock](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-ec2-ntp.html)
- [EC2 single-flow networking limits](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html)
- [EC2 placement-group strategies and limitations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-strategies.html)
- [EC2 On-Demand Capacity Reservations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)
- [Capacity Blocks for ML: supported instance types and Regions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html)
- [AWS Transit Gateway multicast and quotas](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html) and [multicast quotas](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-quotas.html)
- [Amazon EKS hardware device management and DRA](https://docs.aws.amazon.com/eks/latest/userguide/device-management.html)
- [AWS EC2 accelerated instance specifications (including G7/G7e GPU counts and sizes)](https://docs.aws.amazon.com/ec2/latest/instancetypes/ac.html)
- [AWS announcement: G7 in additional Regions, including London](https://aws.amazon.com/about-aws/whats-new/2026/11/amazon-g7-additional-regions/)
- [AWS announcement: G7e in London](https://aws.amazon.com/about-aws/whats-new/2026/05/amazon-ec2-g7e-london-region/)
- [AWS announcement: G7e in Frankfurt, Stockholm and Mumbai](https://aws.amazon.com/about-aws/whats-new/2026/07/amazon-g7e-additional-regions/)
- [NVIDIA Video Encode/Decode Support Matrix](https://developer.nvidia.com/video-encode-decode-support-matrix)
- [NVIDIA MIG supported GPU list](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/supported-gpus.html)

---

## 1. Domain primer: what actually has to happen, per frame

This section exists because every architectural decision below is a consequence of these steps.

### 1.1 The physical/commercial reality

- LED perimeter boards around the pitch carry physically sold sponsors (local/league/club).
- Virtual replacement inserts different sponsors per broadcast territory, invisible in the stadium.
- Contracts are typically sold as **exposure minutes per board slot per territory**, verified by third-party measurement. Therefore **proof-of-play is a contractual artefact, not a nice-to-have log**.
- Territorial rules differ: gambling/alcohol restrictions, league vs club sponsor conflicts, competition-owner approvals, markets where replacement is forbidden. This is a **per-market policy engine**, not a rendering detail.

### 1.2 The per-frame processing chain (logical blocks)

```
[1] Ingest/depayload   : SRT/RIST/RTP/Zixi transport or SDI/IP capture → depacketise/demux → codec-specific decode → GPU surface (NV12/P010 for 4:2:0; a 4:2:2-capable surface format where the decoder supports 4:2:2; v210 is the packed baseband interface format, not a typical decoder output) [V]; use NVDEC only where the exact codec/profile/chroma/bit-depth appears in NVIDIA's support matrix; JPEG 2000/JPEG XS require a separately qualified decoder/SDK or hardware path
[2] Shot/cut detect    : hard cut, dissolve, wipe, DVE, replay transition, graphics overlay
[3] Camera re-ID       : match current shot to a calibrated camera fingerprint (main, 16m, 6m, reverse, beauty…)
[4] Field registration : pitch line/arc detection → homography H (image ↔ pitch model), lens/zoom state
[5] Board geometry     : per-board quads, slot IDs, mapping physical board metres → ad slots
[6] Segmentation       : boards vs pitch vs players vs staff vs objects (semantic + instance)
[7] Occlusion matting  : sub-pixel alpha at board edges; players/legs/ball/flags IN FRONT of boards
[8] Temporal tracking  : optical flow / tracker to keep matte stable frame-to-frame (anti-flicker)
[9] Render per market  : perspective warp ad texture → defocus + motion blur + grain + colour/brightness match
[10] Composite         : alpha blend behind occluders, respect on-screen graphics/scorebug
[11] QC gate           : temporal consistency, geometry sanity, "is the ad actually there?" check
[12] Encode per market : NVENC only for supported codec/profile/chroma/bit-depth combinations and presets; JPEG 2000/JPEG XS require a separate codec implementation/SDK or hardware encoder
[13] Egress            : SRT/RIST/Zixi/JPEG XS/JPEG 2000 as required → broadcaster, with audio + ANC delayed identically
```

**Critical structural consequence:** steps [2]–[8] are **market-independent**. Steps [9]–[13] are **per-market**.

⇒ **Analyse once per match where the source and camera path are shared, then render/encode K outputs.** Cost and GPU demand can scale sub-linearly with the number of territories. This is not universal if markets use different source feeds, policy-dependent masks, graphics paths or processing variants; measure and model those exceptions. This principle should drive pod layout, bin-packing and the business case.

**Second consequence (to be benchmarked, not assumed):** the per-market cost includes compositing and encoding. A simple warp/blend may be inexpensive, but multiple masks, motion blur, colour transforms, HDR conversion, 4K output or memory copies can make compositing material. Hardware encoding may be the bottleneck only when the selected output codec/profile is supported by NVENC. In particular, NVIDIA's current matrix lists no H.264/HEVC 4:2:2 encode support for A10, L4 or L40S, while RTX PRO 4500/6000 Blackwell supports 4:2:2 encode. Therefore the required 10-bit 4:2:2 output format is a first-order instance-selection gate, not a late benchmark detail. [V]

### 1.3 Bandwidth arithmetic (derive, don't guess)
| Format | Active picture | Practical container | Per frame (v210, 4:2:2 10‑bit) |
| --- | --- | --- | --- |
| 1080i50 / 1080p25 | 1.037 Gb/s | HD‑SDI 1.485 Gb/s; ST 2110‑20 ~1.1–1.3 Gb/s | 5.53 MB |
| 1080p50 | 2.074 Gb/s | 3G‑SDI 2.97 Gb/s; ST 2110‑20 ~2.2–2.5 Gb/s | 5.53 MB × 50 = 276 MB/s |
| 2160p50 | 8.294 Gb/s | 12G‑SDI 11.88 Gb/s; ST 2110‑20 ~8.8–9.5 Gb/s | 22.1 MB × 50 = 1.11 GB/s |

v210 packs 6 pixels into 128 bits ⇒ 5120 bytes/line × 1080 = **5,529,600 B/frame**.

Compressed contribution/mezzanine, typical ranges:

- **JPEG 2000 HD** visually lossless ~100–250 Mb/s, latency ~1 frame (sub-frame possible with slice-based profiles).
- **JPEG XS (ST 2110-22)** HD ~100–250 Mb/s at 4:1–10:1, latency sub-frame (lines, ~µs–ms).
- **H.264/HEVC mezzanine** 15–50 Mb/s HD, latency depends on GOP/lookahead/low-delay settings (tens of ms to >100 ms).

Consequences you cannot escape:

- A 2-second uncompressed 1080p50 v210 ring buffer is **~553 MB**. Keep the active low-latency ring buffer in bounded host-pinned or GPU memory; use NVMe/object storage for asynchronous recording or diagnostics, not as the real-time working buffer.
- Uncompressed 1080p50 is about 995 GB/hour (276.48 MB/s × 3,600 s). At an illustrative cross-AZ rate of ~$0.01/GB in each direction (~$0.02/GB counting both sides), moving one such stream across AZs is roughly **$20/hour per stream**, before any additional service charges. NAT processing can add material cost; exact rates vary by region and path. **Recommendation:** keep decode/analysis/composite/encode node-local where possible. If multiple GPUs are required, keep them on one node and explicitly benchmark PCIe/GPU-P2P transfers; do not send raw frames across AZs, and do not make inter-node raw-frame transport the default design.
- Compressed 20 Mb/s egress to the public internet is about 9 GB/hour, or ~$0.8/hour per market at an illustrative $0.09/GB. Ten markets × 2.5 hours is about **$20/match** on that assumed rate. The actual bill depends on destination, region, tiering and whether delivery uses internet egress, Direct Connect or a managed media service. Egress is material, but GPU-hours may still dominate.

### 1.4 Interlace, graphics and replays — the three things that break naive pipelines

- **1080i50:** 50 fields/s are captured 20 ms apart and paired into 25 interlaced frames/s. The two fields can contain different motion positions. Treating an interlaced frame as a progressive image can create combing and unstable board/player edges. Options include field-aware processing at 50 fields/s, motion-adaptive deinterlace → process → reinterlace, or conversion upstream. Each changes latency, vertical detail and inference workload; test on representative pans and fast player motion.
- **On-screen graphics** (scorebug, lower thirds, VAR lines, replay wipes, split screens) are in front of the world feed. Your composite must not paint over them and must not treat them as pitch/boards. A DVE/split-screen replay can contain two different camera geometries in one frame.
- **Replays:** if you process only PGM, replays are processed automatically but arrive as slow motion / different camera / possibly different speed ramps — temporal trackers must handle 0.25×–3× and reversed playback. If you process ISOs and re-switch, replay sources (EVS) are separate feeds you must also cover, otherwise a replay shows the original physical ads — a contractual breach.

### 1.5 Latency budget — worked example (1080p50, continental)
| Stage | Typical | Note |
| --- | --- | --- |
| Contribution encode (J2K) | 20–40 ms | ~1–2 frames; XS: <5 ms |
| Fibre propagation (600 km) | 3 ms | physics |
| Network + jitter/retransmit buffer | 60–150 ms | SRT latency is configured above the path RTT/jitter; several times RTT is only a planning heuristic, not a protocol guarantee |
| Depayload + codec-specific decode | 5–15 ms | Illustrative; J2K/XS decode is not NVDEC and must be benchmarked on its actual implementation |
| Cut detect + registration + segmentation + matting | 10–30 ms | [A] benchmark required |
| Composite (per market, GPU) | 0.5–2 ms |  |
| Hardware encode (e.g. NVENC where supported) | 5–15 ms | Illustrative; output codec/profile must be supported; lookahead/B‑frames add frames |
| Egress transport buffer | 60–150 ms | again sized from the destination RTT and measured jitter/loss (same heuristic caveat as above) |
| Downstream decode/frame sync | 20–40 ms |  |
| Total for the stated J2K example | ~185–445 ms | ~9–22 progressive frames at 50 fps; field/frame convention must be explicit |

Add for reality:

- **Satellite contribution:** budget approximately +240 ms propagation for a GEO path, plus modem, FEC, interleaver and receiver buffering. A practical one-way addition can be roughly +250–350 ms, depending on equipment and configuration.
- **Intercontinental processing** (e.g. Europe stadium → us-east → Europe broadcaster): +70–110 ms of pure propagation (about one extra transatlantic round trip at ~5 µs/km; measure the real path) [V], plus two transport buffers sized on a much larger RTT. This is why "process in the region closest to the source, fan out from there" usually beats "process near each market".
- **OTT distribution** (HLS/DASH/CMAF): +2–30 s. That is a different budget; do not confuse contribution latency with viewer latency.

**Logical consequence for the AWS design:** minimise total path length **stadium → compute → each broadcaster**, not "distance to the nearest AWS region". Hairpinning (stadium → region A → back to continent B) is the thing to eliminate.

### 1.6 Clock discipline (the most underestimated problem)

If your ingest and egress clocks differ by 100 ppm, over a 2-hour match the relative drift is 0.72 s → buffer underflow/overflow → dropped or repeated media units. On prem, a broadcast plant typically disciplines clocks with PTP/genlock. EC2 does not provide a replacement broadcast genlock chain, so you must:

- **preserve source media timestamps, PTS/DTS and embedded timecode at ingest**, estimate the source media clock, and schedule egress against that media timeline. Do not treat EC2 wall-clock NTP/PHC as a substitute for source programme time. Apply asynchronous sample-rate conversion for audio and an explicit, audited frame/field drop-repeat policy for video;
- **keep audio and video in the same delay line** so lip-sync and commentary alignment survive;
- **preserve or re-insert ANC** (ATC timecode carrying LTC/VITC, AFD, closed captions, SCTE-104/35, Dolby metadata). Many IP/cloud paths silently drop ANC. Proof-of-play must be referenced to broadcast timecode, not wall clock — so if ANC is lost you must embed your own reference (e.g. SEI) and carry a mapping.

---

## 2. CURRENT ARCHITECTURE — own data centre, GPU park, Kubernetes

### 2.1 End-to-end signal flow

The following is an illustrative baseline, not a verified inventory of the existing facility. Confirm the actual ingest/egress gateways, codec licences, GPU placement, PTP topology, fail-to-clean implementation and per-match redundancy before treating it as an as-built diagram.

```
┌──────────────────────────── STADIUM / OB COMPOUND ───────────────────────────┐
│ Cameras 1..30 → CCU → vision mixer → PGM "world feed" (dirty)                │
│ LED perimeter boards (physical sponsors)   [optional IR/key overlay system]  │
│ Audio: intl sound + commentary stems (± Dolby E)   ANC: TC/AFD/SCTE‑104      │
│ In‑venue big screen & stadium radio: MUST remain physical (never replaced)   │
│                                                                              │
│ Contribution A: fibre, J2K/XS or SDI handoff   ┐ dual path                   │
│ Contribution B: satellite DVB‑S2 or bonded 5G  ┘ (2022‑7 / SRT bonding)      │
└──────────────┬───────────────────────────────────────────────┬───────────────┘
               ▼                                               ▼
┌─────────── DC MEET‑ME / EDGE ROOM ───────────────────────────────────────────┐
│ Teleport: dishes, L‑band → IRD, LNB/BUC, redundancy                          │
│ Fibre handoffs / carrier cross‑connects, DWDM if you own wavelengths         │
│ SDI/IP gateway, frame synchronisers, PTP GM (ST 2059‑2) + black/tri‑level    │
│ SMPTE 2022‑7 seamless merge, embed/de‑embed, ANC handling, loudness bypass   │
│ ★ FAIL‑TO‑CLEAN bypass switch (hardware, matched delay)                      │
└──────────────┬───────────────────────────────────────────────────────────────┘
               ▼  ST 2110 / 2022‑6 / SDI
┌─────────── DC MEDIA FABRIC (spine–leaf 25/100G) ─────────────────────────────┐
│ VLAN/VRF separation: MEDIA | CONTROL | STORAGE | MGMT | OOB                  │
│ IGMP/PIM multicast, PTP boundary/transparent clocks, ST 2110‑21 traffic shape│
│ Leaf per rack, non‑blocking or 3:1 with headroom for uncompressed bursts     │
└──────────────┬───────────────────────────────────────────────────────────────┘
               ▼
┌─────────── KUBERNETES CLUSTER(S) ────────────────────────────────────────────┐
│ Ingest nodes  : SDI capture cards / Rivermax+ConnectX / DPDK, HW timestamp   │
│ GPU media pods: per‑match "vertical slice" (see 2.4)                         │
│ Egress nodes  : J2K/XS/SRT/RIST encap, 2022‑7 dual path                      │
│ Control plane : match scheduler, ad playlist, calibration, model registry    │
│ Platform      : GPU Operator, CNI, CSI, registry, Prometheus, Loki, Argo CD  │
└──────────────┬───────────────────────────────────────────────────────────────┘
               ▼
   Per‑market handoff → broadcaster MCR / regional production centre / OTT origin
               ▼
   Downstream: playout, local graphics/commentary, distribution
```
### 2.2 Facility & hardware layer
| Layer | Content | Engineering constraints |
| --- | --- | --- |
| Power | Dual utility feeds, UPS (N+1), diesel gensets, ATS, PDU A/B | GPU racks are power‑dense; air‑cooled racks historically ~5–15 kW, modern 8×GPU nodes can exceed that per node‑group. Density, not floor space, is the limit. |
| Cooling | CRAC/CRAH, hot/cold aisle containment, or rear‑door/DLC for high density | Thermal throttling directly causes inference jitter → late frames. |
| Compute | GPU nodes (inference/graphics class), CPU nodes (control, ingest, storage), NVMe scratch | Consumer‑vs‑datacenter GPU encode session limits; ECC; clock locking for determinism |
| Network | Leaf/spine, 25/100G, PTP‑aware switches, multicast | ST 2110‑21 sender timing compliance; microburst loss on uncompressed video is unforgiving |
| Timing | 2× PTP grandmasters (GNSS), boundary clocks, genlock distribution | GM failover is a live‑on‑air event |
| Storage | Local NVMe (scratch/models), shared NFS/Ceph/Lustre (assets), object store (archive) | Model/engine load time is a cold‑start problem (§2.7) |
| Broadcast glue | SDI routers, frame syncs, embedders, 2022‑7 mergers, bypass relays | This layer has no cloud equivalent |
| Out‑of‑band | Serial console, IPMI/Redfish, KVM, separate mgmt network | Remote hands dependency |

### 2.3 Kubernetes platform (self-managed)

- **Control plane:** 3× etcd + apiserver/scheduler/controller-manager, HA VIP, own PKI, own backup/restore, own upgrade cadence. You own etcd defrag, compaction, certificate rotation, and the blast radius of a bad admission webhook.
- **CNI:** Calico/Cilium (BGP or overlay) for control; media plane bypasses CNI via Multus + SR-IOV/macvlan/host-device or `hostNetwork`, because you need multicast, specific NIC queues, jumbo frames, and no encapsulation overhead.
- **CSI:** local-path/NVMe for scratch, NFS/Ceph for shared assets.
- **GPU:** NVIDIA driver + container toolkit + device plugin (or GPU Operator), DCGM exporter, `nvidia-persistenced`, locked application clocks, exclusive compute mode for live pods; time-slicing/MPS only on offline/QA nodes (see DC-11).
- **Node tuning:** CPU Manager `static`, Topology Manager `single-numa-node`, Memory Manager `static`, Guaranteed QoS, hugepages, IRQ affinity away from media cores, `isolcpus`/`nohz_full` on media cores, tuned low-latency profile, CFS quota avoidance.
- **Why:** CFS quota/period throttling can create scheduling stalls; a 100 ms stall spans five 20 ms frame/field deadlines. Treat it as one plausible source of jitter alongside GPU contention, queueing, I/O, NIC pressure, thermal throttling and source-network variation. Measure CPU throttling and scheduling delay per media pod rather than assuming it is the root cause.
- **Delivery:** GitOps (Argo CD/Flux), Helm/Kustomize, internal registry with pre-pulled images, PodDisruptionBudgets, priority classes, match-aware drain locks.

### 2.4 The media pod: "vertical slice", not microservices

```
┌─ POD: match-<id> (1 node, 1..n GPUs, pinned cores) ──────────────────────────────────────────┐
│ hostNetwork or Multus/SR-IOV VF (dedicated NIC), hugepages, Guaranteed QoS                   │
│                                                                                              │
│ [ingest]   SRT/RIST/2110 → depacketise/jitter buffer → codec-specific decoder / raw surface  │
│                                            ▼                                                 │
│ [analysis] cut detect → camera re-ID → homography → segmentation → matting → track           │
│            (ONE shared analysis pass per match where inputs permit)                          │
│                                            ▼                                                 │
│ [render]   market 1 | market 2 | … | market K : warp/blur/grain/alpha on GPU                 │
│                                            ▼                                                 │
│ [encode]   supported NVENC paths and/or J2K/XS codec workers; GPU-local or same-node         │
│            transfers only, with measured PCIe/P2P cost                                       │
│                                            ▼                                                 │
│ [egress]   SRT/RIST per market, audio + ANC in the same delay line                           │
│                                                                                              │
│ sidecar-free: no service mesh, no per-stage network hops, no serialisation                   │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```
**Rationale:** every stage boundary turned into a network hop adds copying/serialization, a queue with tail latency, and a failure domain. One 1080p50 v210 stream is about 276 MB/s. Keep the real-time processing path in one pod and on one node; use one GPU only where measured capacity permits, otherwise use multiple GPUs on that node with an explicitly measured GPU-P2P/PCIe or shared-memory path. Microservices remain appropriate for the control plane, not for per-frame orchestration across a network.

### 2.5 Control plane (this part is microservices)
| Service | Responsibility | Store |
| --- | --- | --- |
| Match calendar & orchestration | Fixtures, kickoff windows, pre‑warm, capacity booking, delays/extra time | RDBMS |
| Calibration service | Per‑stadium/per‑camera geometry, lens tables, board slot maps, versioning | RDBMS + object store |
| Ad inventory & scheduling | Contracts, rotations, territory policy, exclusivity, make‑goods | RDBMS |
| Asset pipeline | Ingest, validate, colour‑convert, pre‑warp/mip‑map, version, distribute | Object store + cache |
| Model registry & release | Training artefacts, TensorRT engines per GPU arch/shape, rollout gates | Object store + registry |
| Rights/policy engine | Which market may replace which board slot with which category | RDBMS, audited |
| Proof‑of‑play | Frame‑accurate visibility/exposure logs, hash‑chained, third‑party export | Columnar/object + warehouse |
| Ops/monitoring | SLOs, per‑frame counters, alarms, runbooks | TSDB + logs |

### 2.6 HA model (current)

- **Per match:** 1+1 hot processing replicas (A/B), preferably in separate racks, power feeds and network failure domains (a single DC has no AZs), both receiving equivalent time-aligned source media. Select between them using a broadcast-aware frame/field switch at the physical edge unless the encoded outputs are demonstrably binary-identical and suitable for packet merge. SMPTE 2022-7 is packet protection for identical redundant streams; it is not a general selector for two independently rendered outputs.
- **Fail-to-clean:** a hardware bypass path carrying the unmodified world feed delayed by the same amount, so switching does not jump time. **Never fail to black, never fail to frozen.**
- **Node/GPU:** N+1 spare capacity, spare GPUs on shelf, spares for NICs/PSUs.
- **Site:** the honest answer for most single-DC operators is **no true site redundancy** — a second DC with genlocked broadcast glue, teleport diversity and duplicate GPUs is a large capex step. This is usually the strongest single argument for the cloud move.

### 2.7 Cold start reality (applies to both architectures)

Time to bring a match pipeline to first correct frame:

```
node ready + driver init      : 20–90 s (bare metal boot can be minutes)
image pull (CUDA+cuDNN+TRT)   : 5–15 GB image; gzip decompress often 100–250 MB/s single-thread
                              → 30–150 s unless pre-cached / lazy-loaded
TensorRT engine load/build    : load 5–30 s from fast NVMe; BUILD from ONNX = minutes
camera calibration load       : seconds
tracker/model warmup          : first inferences are slower (caches, autotune)
```

**Consequence:** you cannot react to "match started" by scaling. **Capacity must exist before kickoff, warmed, with engines pre-built per GPU architecture and input shape.** On prem you do this by over-provisioning; on AWS you do it with reservations + scheduled pre-warming (§3.10).

### 2.8 Utilisation & cost shape (why you are considering the move)

Football demand is extremely peaky [A]:

- **Sat 15:00/15:30 CET:** several concurrent domestic matches.
- **Midweek European competition:** a slot with many concurrent matches (the current league-phase formats concentrate ~9 fixtures per evening, split across two kickoff times) [V: exact competition format/slots per season].
- **Otherwise:** near-zero live load (only training, asset prep, catch-up re-renders, QA).

**A peak-to-mean ratio of 5–15× is an illustrative planning assumption [A], not a universal industry constant.** Calculate the ratio from the actual fixture calendar and offline workload. A DC sized for peak can be idle much of the week; a DC sized for mean cannot deliver peak. That asymmetry, plus single-site risk and hardware lead time, may form the business case — not an assumption that cloud is cheaper per GPU-hour (it may not be at high utilisation).

TCO comparison must be like-for-like:

```
DC $/match = (capex amortisation + power/cooling + space + network/transit + teleport/satellite
              + software/licences + spares + staff + refresh risk) / (matches delivered at peak SLA)

Cloud $/match = GPU-hours (incl. HA + idle reserve + reservations) + transport + egress
              + storage + platform + observability + staff
```

**Include the idle reserve you must still pay for in the cloud** (reservations, warm standbys) or the comparison is dishonest.

---

## 3. TARGET ARCHITECTURE — AWS + EKS

### 3.1 Design principles, each derived from a constraint above
| # | Principle | Derived from |
| --- | --- | --- |
| P1 | Use compressed contribution where appropriate (SRT/RIST/Zixi, JPEG XS/J2K); do not assume an AWS VPC is a drop-in ST 2110-20 broadcast fabric. | Limited TGW multicast; no genlock; no broadcast timing plant; transfer cost |
| P2 | One match = one scheduling/failure unit. Keep the frame path in one pod and on one node; use one GPU if measured capacity allows, otherwise multiple GPUs in the same node with explicit GPU-local transfers. | 276 MB/s raw at 1080p50; tail latency; cross-AZ cost |
| P3 | Analyse once, render/encode K times. | Market‑independent CV stages |
| P4 | Keep each replica's media path AZ-local; run A/B replicas in separate AZs, each self-contained. | Cross-AZ cost + latency + failure independence |
| P5 | Capacity reserved and pre‑warmed on the fixture calendar, not autoscaled reactively. | Cold start ≫ kickoff tolerance; GPU capacity scarcity |
| P6 | Deterministic, identical A/B processing (same GPU model, same engine, deterministic kernels). | Frame‑accurate seamless switching |
| P7 | Fail‑to‑clean, delay‑matched, at a physical edge you still control. | Broadcast obligation; cloud has no SDI/genlock |
| P8 | Control plane cloud‑native and elastic; media plane pinned and static during a match. | Different scaling/consistency needs |
| P9 | Keep a physical/timing island (stadium, OB, teleport, MCR handoff) on prem or at a partner facility. | Physics and SMPTE reality |
| P10 | Semantic monitoring, not just process monitoring. | A silently empty matte looks like a healthy feed but breaches contracts |

### 3.2 Topology options — choose deliberately

**Option A — Centralised cloud rendering** (closest to today's model)

- Stadium → contribution → one AWS region → K rendered market feeds → K broadcasters.
- Simple, one place to operate, highest egress and longest paths for distant markets.

**Option B — Multi-region, source-local processing**

- Process in the region nearest the contribution source/IBC; fan out per-market from there. Optionally a second region for markets on another continent, receiving one clean feed + metadata.
- Minimises total path; multiplies platform footprint and capacity reservations.

**Option C — Central analysis, distributed rendering ("metadata + clean feed")**

- Cloud runs the shared analysis stages [2]–[8] once and distributes a time-aligned source picture plus the data needed to reproduce the composite at each edge. The interface must be explicit: either the original PGM plus board geometry and foreground/occlusion mattes, or a clean plate plus foreground/occlusion layers, slot IDs and a versioned playlist. A clean feed without the foreground/occlusion information is not sufficient to preserve players and staff in front of the boards.
- This can reduce cloud encoding from K outputs to one source feed plus compact metadata, but every edge site still needs a renderer, encoder, synchronised media clock, and fail-to-clean path. The bandwidth benefit depends on the number and geography of edge sites; it is not automatically one egress stream for the entire service. Costs include edge fleet operations, secure clean-feed handling, monitoring, synchronisation and ownership of the matte/metadata interface. It is attractive where renderers can be placed at broadcaster facilities, but requires a separately validated product and operational model.

**Option D — Hybrid island (recommended transitional and probably permanent)**

- Physical/timing edge stays on prem; cloud does analysis + rendering; a small on-prem footprint keeps SDI/genlock/2022-7/bypass and acts as the last-resort clean path.

| AWS placement option | What it can realistically do here | Caveats |
| --- | --- | --- |
| Region (EKS + G‑family EC2) | Bulk analysis + rendering, K markets | Propagation delay; GPU capacity; no genlock |
| Local Zones | Metro‑close rendering (Option C) | Limited instance families (historically T4/G4dn class in many metros) [V]; fewer services; parent‑region control‑plane dependency |
| Outposts (rack) | GPU compute inside your DC/IBC/broadcaster with AWS APIs | GPU families historically limited (T4/G4dn class) [V]; control plane lives in the parent region → network partition means running pods continue but no rescheduling; capacity is a fixed physical asset (no elasticity) |
| Wavelength / carrier edge | Ultra‑close to a 5G contribution path | Very location/carrier specific; ecosystem volatile [V] |
| EKS Hybrid Nodes | On‑prem/edge nodes joined to an EKS control plane, one GitOps/API surface | Requires reliable DX/VPN/internet to the region; OS and feature constraints; GPU support scope [V] |
| EKS Anywhere / self‑managed K8s on prem | Modernise the DC leg without pretending it's cloud | You still own the control plane |

### 3.3 Reference AWS architecture (Option A/D baseline, with C as an evolution)

```text
STADIUM / OB / TELEPORT (physical timing island)
 SDI/2110 plant, PTP GM, genlock, 2022-7 merge, ★ delay-matched fail-to-clean
        │
        ├── Contribution path A: compressed SRT / supported RIST / Zixi / JPEG XS
        └── Contribution path B: diverse compressed path, with source timestamps and ANC
                    │
                    ▼
      AWS transport ingress (MediaConnect only for supported profiles,
      or self-managed protocol gateways where codec/profile/encryption demands it)
                    │
             duplicate to each AZ
          ┌─────────┴───────────────────────────────────────┐
          ▼                                                 ▼
┌──────────────────── AZ-A ────────────────────┐  ┌──────────────────── AZ-B ────────────────────┐
│ EKS media NodePool; exact GPU SKU reserved   │  │ EKS media NodePool; same GPU SKU reserved    │
│                                              │  │                                              │
│ match pod A (1..n GPUs, one node)            │  │ match pod B (1..n GPUs, one node)            │
│ codec-specific decode → shared analysis      │  │ codec-specific decode → shared analysis      │
│ → render K markets → supported encoders      │  │ → render K markets → supported encoders      │
│ → market outputs A                           │  │ → market outputs B                           │
│                                              │  │                                              │
│ local NVMe cache: engines, textures, models  │  │ local NVMe cache: engines, textures, models  │
└──────────────────────┬───────────────────────┘  └──────────────────────┬───────────────────────┘
                       │                                                  │
                       └──────────────────┬───────────────────────────────┘
                                          ▼
                 PHYSICAL BROADCAST EDGE / MCR HANDOFF
       frame/field-aware A/B selector + delay-matched clean-feed input
       audio/ANC/timecode alignment, frame sync, genlock, final playout

CONTROL PLANE (multi-AZ; no raw frames)
 match orchestration | playlist/policy | calibration | model registry | proof-of-play
 DynamoDB/RDS | S3/ECR | EventBridge Scheduler/Step Functions | observability

OFFLINE (no live SLA)
 model training, engine builds, recorded-match regression and catch-up re-renders
```

The two live replicas receive equivalent source media over independent paths and process independently in separate AZs. Keep each replica's raw-frame path inside its node; do not use cross-AZ frame transport to share analysis state. The diagram is a logical pattern: confirm the number of ingest copies, exact MediaConnect protocol/profile, GPU count per node, output fan-out and edge selector topology against the chosen AWS Region and broadcaster handoff.

### 3.4 Component mapping: current → target
| Current | AWS target | Notes / traps |
| --- | --- | --- |
| Own K8s control plane | EKS (multi‑AZ, managed) | You still own node lifecycle, GPU stack, CNI config, upgrades |
| Bare‑metal GPU nodes | EC2 G7/G7e (RTX PRO 4500/6000 Blackwell), plus G6/G6e/G5 candidates [V availability per AZ] | Blackwell candidates add H.264/HEVC 4:2:2 NVENC/NVDEC support; A10/L4/L40S do not provide 4:2:2 NVENC. P4/P5/P6 accelerators are not default unified-pipeline choices because their listed data-centre GPUs have no NVENC; they may still be considered for analysis-only workloads with separate encoding. |
| Cluster Autoscaler | Karpenter + static NodePools for media | Consolidation must be disabled during matches |
| NVIDIA driver/toolkit manual | GPU Operator on EKS accelerated AMIs (AL2023/Bottlerocket) | Driver ↔ CUDA ↔ TensorRT compatibility matrix is a hard constraint; do not double-install the driver/toolkit if the AMI already ships them (disable the duplicate operator components) [V] |
| Local NVMe scratch | Instance store NVMe | Lost on stop/terminate → cache only, never state |
| NFS/Ceph assets | S3 (+ FSx for Lustre / EFS only if POSIX semantics are truly needed) | S3 GET latency vs. asset hot‑reload; pre‑fetch to NVMe |
| Object archive | S3 Glacier tiers | Retrieval time/cost; data gravity on exit |
| SDI/IP gateway, 2022‑7 merge | MediaConnect or self‑managed SRT/RIST + on‑prem merge | 2022‑7 merge is a physical/timing function; keep it at the edge |
| PTP GM / genlock | Amazon Time Sync NTP; PHC on supported Linux EC2 instances in `precision-time` placement groups; timestamp-based media-clock recovery | Microsecond-class PHC is not a broadcast ST 2059-2 grandmaster, tri-level sync or genlock replacement |
| Teleport / satellite | Partner teleport + Direct Connect | You cannot put a dish in a VPC |
| Hardware bypass relay | Stays physical at edge | Cloud cannot fail‑to‑clean a signal it isn't carrying |
| RDBMS | Aurora / RDS Multi‑AZ |  |
| Redis | ElastiCache | Not in the frame path |
| Kafka telemetry | MSK / Kinesis | Never per‑frame per‑market events at full rate (see §3.11) |
| Prometheus/Grafana/Loki | AMP / AMG / S3+OpenSearch | Watch cardinality cost |
| Vault | Secrets Manager + External Secrets / CSI driver, KMS |  |
| Registry | ECR + SOCI/lazy loading or AMI‑baked images | Cold start |
| CI engine builds | CodeBuild/CodePipeline/GitHub Actions on GPU runners | One engine per (GPU arch × shape profile × precision) |
| Batch training | SageMaker / AWS Batch / EKS + Volcano/Kueue on Spot | Spot fine offline, never live |

### 3.5 Transport decision table
| Transport | Latency | Loss recovery | AWS fit | Use |
| --- | --- | --- | --- | --- |
| SRT | configurable; commonly planned at several × RTT (heuristic, not a guarantee) | ARQ | Works over internet/NLB/GA; MediaConnect supports it (SRT sources: failover mode only, no merge) | Default contribution |
| RIST | depends on RTT/jitter and configured recovery window | ARQ; profile-dependent | MediaConnect currently supports **Simple profile only** and does not provide RIST encryption | Alternative where Simple profile is acceptable; use a qualified gateway for Main-profile requirements |
| Zixi | similar | ARQ, FEC | MediaConnect supports Zixi push sources (Zixi pull cannot be a source); used by many broadcasters | Broadcaster interop |
| JPEG XS | sub-frame codec latency is possible; transport/network buffering is additional | No ARQ inherent to the codec | MediaConnect supports ST 2110 JPEG XS source/output flows; confirm region, flow limits and exact profile; ST 2110 JPEG XS sources are single-source (no MediaConnect source failover), so duplicate at the sender or use separate flows | Low-latency compressed contribution/egress where supported |
| JPEG 2000 | ~1 frame | none | Self‑managed encode/decode on EC2/EKS; SDK licensing | Legacy contribution |
| ST 2110-20 uncompressed | very low media-interface latency | none | Not a drop-in VPC fabric; Transit Gateway multicast is limited (documented non-adjustable 1 Gb/s per flow and 20 Gb/s aggregate per AZ; a single 1080i50/p50 ST 2110-20 flow exceeds the per-flow limit) and is not routed over Direct Connect | Keep in the physical broadcast island unless a specific cloud design is independently proven |
| ST 2022‑6 | ~0 | 2022‑7 protection | Unicast possible but bandwidth‑hungry | Edge only |
| HLS/DASH/CMAF | 2–30 s | TCP | MediaPackage/CloudFront | OTT distribution, not contribution |

**Per-flow bandwidth caveat:** AWS documents a 5 Gb/s single-flow limit when two instances are not in the same cluster placement group; a cluster placement group can raise this to 10 Gb/s. ENA Express can provide up to 25 Gb/s between eligible instances in the same AZ. These are instance-to-instance limits, not guarantees for public internet, MediaConnect or every network path. Validate packet size, UDP behaviour, sustained bandwidth and instance-size baselines for the actual topology. [V]

**Placement-group and reservation trade-off:** a rack-level spread placement group supports at most seven running instances per AZ per group. EC2 On-Demand Capacity Reservations can be used with cluster placement groups, but not with spread or partition placement groups. Cluster placement improves intra-group latency/throughput but is AZ-local and may correlate hardware placement. Default placement is often a reasonable starting point for independent nodes; use cluster placement only when measured intra-node/cluster traffic benefits justify it. Keep A/B replicas in separate AZs. Do not combine a spread-placement recommendation with an ODCR design without checking this incompatibility.

### 3.6 EKS design specifics that actually matter here

**Node pools (separate them, they have opposite requirements):**

| NodePool | Instance types | Capacity mode | Disruption |
| --- | --- | --- | --- |
| media-live | G7/G7e first candidates where 4:2:2 is required; G6/G6e/G5 only where codec requirements fit | Zonal ODCR for the exact instance type/AZ, plus on-demand fallback only as best-effort [V] | Blocked during match windows (`karpenter.sh/do-not-disrupt`, NodePool disruption budgets with match-window schedules, and external drain locks) |
| media-warm | same SKU as the paired live pipeline | On-demand, pre-provisioned at T-45 min or earlier | blocked during window |
| control | M/C families, multi‑AZ | on‑demand + Compute Savings Plan | normal, PDBs |
| offline-gpu | G/P | Spot | aggressive, checkpointed |
| edge (optional) | Local Zone / Outposts / hybrid nodes | fixed | manual |

**Pod spec essentials (illustrative; confirm CPU/memory/GPU topology on the selected instance):**

```yaml
spec:
  priorityClassName: live-critical
  terminationGracePeriodSeconds: 120  # controlled shutdown only after failover/bypass is engaged
  hostNetwork: true                   # optional; benchmark against the chosen VPC CNI/Multus design
  containers:
    - name: match-pipeline
      resources:
        requests:
          cpu: "16"
          memory: "32Gi"
          nvidia.com/gpu: "1"         # request more GPUs if the measured match slice needs them
          hugepages-2Mi: "2Gi"         # only where the media stack uses hugepages
        limits:
          cpu: "16"
          memory: "32Gi"
          nvidia.com/gpu: "1"
          hugepages-2Mi: "2Gi"
```

For Guaranteed QoS, CPU and memory requests must equal limits for every container in the pod. Configure CPU Manager `static`, Topology Manager and NUMA placement on the node; ensure all containers and any injected sidecars are accounted for. `terminationGracePeriodSeconds` is not a match-duration drain strategy: prevent voluntary disruption during a live match, engage the clean bypass or standby path first, then terminate only after the external controller authorises it.

**Probe design is a real trap:** a Kubernetes liveness probe that fails during a 200 ms GC/IO stall will restart your on-air pipeline. Prefer: readiness gates for admission, semantic health → bypass switch for degradation, and restart only as a last resort with the bypass already engaged.

**GPU sharing and isolation:** G5/A10G, G6/L4 and G6e/L40S do not provide MIG. NVIDIA lists the Blackwell RTX PRO 4500 and RTX PRO 6000 GPUs used by EC2 G7/G7e as MIG-capable; confirm MIG enablement and profile support on the exact EC2 host/driver before relying on it. MIG provides hardware partitions, but validate which NVENC/NVDEC engines and performance are visible to each MIG profile. Time-slicing and MPS are sharing/concurrency mechanisms, not equivalent to MIG fault or memory isolation. Kubernetes DRA is a device allocation interface; it can request devices and supported partitions but does not create the isolation itself. EKS documents DRA support from Kubernetes 1.33 and recommends 1.34+; validate the NVIDIA DRA driver/Karpenter version combination. For critical live media, prefer exclusive GPUs or qualified MIG partitions; do not use time-slicing as a substitute for fault isolation.

**Cold start engineering:**

- **Pre-build TensorRT engines in CI** per (GPU arch, input shape/optimisation profile, precision); store in S3/ECR/AMI; **never build engines at pod start**.
- **Bake or lazy-load images:** SOCI (seekable OCI) with containerd, or AMI-baked images, or zstd compression; a 12 GB CUDA image with single-threaded gzip decompression adds roughly 50–120 s at 100–250 MB/s (§2.7), before network pull time.
- **Pre-pull via DaemonSet** on warm nodes; keep a balloon/pause-pod holding reserved capacity so real pods schedule instantly.
- **Pre-warm ad textures and calibration into NVMe before kickoff.**
- **Quotas/capacity to validate weeks before the season:** On-Demand G/VT vCPU quotas, EC2 capacity reservation limits, EKS cluster/service quotas, subnet IP and ENI capacity, Elastic IP quotas, MediaConnect flows/outputs/entitlements, Direct Connect port capacity, and EBS throughput/IOPS where used. Check service quotas and account-specific limits rather than assuming all are adjustable in the same way. **Quota and AZ capacity work is a project dependency, not an afterthought.**
- **Networking hygiene:** VPC endpoints for S3/ECR/STS/CloudWatch so you don't push asset traffic through NAT (cost + latency + bandwidth ceiling); no service mesh on the media plane; network policies on the control plane; IMDSv2 hop-limit/Pod Identity handled correctly.

### 3.7 Instance selection logic (not a shopping list)

Selection is driven by, in order:

1. **Codec/chroma/bit-depth compatibility first.** If the contribution or output is H.264/HEVC 10-bit 4:2:2, confirm hardware decode and encode support in the exact GPU. The current NVIDIA matrix lists 4:2:2 H.264/HEVC encode on Blackwell RTX PRO 4500/6000, but not on A10, L4 or L40S. Blackwell RTX PRO 4500/6000 also list 4:2:2 decode support. JPEG 2000 and JPEG XS are not NVDEC/NVENC codecs; they require a separate decoder/encoder implementation.
2. **NVENC/NVDEC engine count and throughput** at the exact preset, resolution, frame rate and codec. The current NVIDIA matrix lists (per GPU) 1 NVENC for A10, 2 for L4, 3 for L40S, 3 for RTX PRO 4500 Blackwell Server Edition and 4 for RTX PRO 6000 Blackwell Server Edition. These counts are not a guaranteed stream-density figure.
3. **Memory bandwidth and GPU compute** for warp/blur/grain/alpha, segmentation and matting at 1080p50 × K. Profile the combined analysis + render workload rather than treating any one spec as the answer.
4. **Sustained network bandwidth** (not peak/burst labels), PCIe/GPU-P2P topology and host-memory copy cost. AWS documents NVIDIA GPUDirect P2P support for multi-GPU G7e instances; confirm it for G7 and for the exact instance size [V].
5. **Availability** in the target region and AZ, capacity-reservation eligibility, and exact instance-size limits.
6. **A/B parity:** same GPU SKU, driver, firmware, CUDA/TensorRT build, model, precision and codec settings in both AZs.

Method, not folklore:

```
Measure per stream on candidate instance:
  t_dec, t_analyse, t_composite(per market), NVENC occupancy%, NVDEC occupancy%,
  SM occupancy%, memory BW%, p99.9 end-to-end pod latency, jitter

render_streams_per_GPU = floor( min(encode_profile_budget, composite_budget, memBW_budget, SM_budget, memory_budget) × target_utilisation )
              # use a measured target (0.6 is an example, not a universal constant)
              # size at p99.9 under the real codec, bitrate, frame rate and scene complexity
```

Consult **NVIDIA's Video Encode/Decode GPU Support Matrix** for authoritative engine counts and codec/chroma/bit-depth support. The matrix changes; re-check it at design freeze and benchmark the EC2 GPU SKU and driver rather than relying on GPU-family names alone [V].

**Do not treat Inf2/Trn1 (Neuron) as a drop-in replacement for the unified media GPU:** they do not provide the NVIDIA NVENC/NVDEC/CUDA graphics path assumed by this design. They may be evaluated for a separately decoupled inference stage, but decoding, compositing and encoding would need their own qualified path.

### 3.8 HA / failover design (the hard part)

**Frame/field arithmetic first:** a 2-hour 50p output contains 360,000 frames; a 1080i50 output contains 360,000 fields (180,000 interlaced frame pairs). At 99.99% good output units, 36 units are bad—about 0.72 s at a 20 ms unit interval. To average fewer than one bad unit per match requires about **99.99972% good units**. Define whether the contractual unit is a progressive frame, an interlaced field or a complete interlaced frame. This availability target requires precomputed standby and frame/field-aware switching; restarts cannot meet it.

Design:

```text
               equivalent source media + source-time identity
                         │                     │
                         ▼                     ▼
        ┌────────────── AZ-A ────────────┐  ┌────────────── AZ-B ────────────┐
        │ match replica A                │  │ match replica B                │
        │ decode → analyse → render × K  │  │ decode → analyse → render × K  │
        │ encode × K → outputs A         │  │ encode × K → outputs B         │
        └───────────────┬────────────────┘  └────────────────┬───────────────┘
                        │                                    │
                        └─────────────────┬──────────────────┘
                                          ▼
                 ┌──── PHYSICAL BROADCAST-EDGE SELECTOR ─────┐
                 │ frame/field-aware A/B selection per market│
                 │ aligned audio, ANC, PTS and timecode      │
                 └─────────────────────┬─────────────────────┘
                                       │
             ★ delay-matched clean world-feed input ─────────┘
                                       ▼
                              broadcaster handoff
```

The selector may operate on baseband frames or on compressed streams only if the encoded outputs are proven compatible for seamless switching. Otherwise, decode/frame-sync/select/re-encode or another broadcast-qualified switching design is required. The selector and clean bypass remain independent of the cloud media plane.

**Three non-obvious consequences:**

1. **Determinism is a requirement, not an optimisation.** If A and B use different GPU architectures, different TensorRT engines, TF32 vs FP32, non-deterministic reductions, or cuDNN autotuning, their mattes differ by sub-pixels → the switch itself becomes a visible flicker. **Pin GPU model, engine, precision, and deterministic algorithm selection; verify with bit/PSNR comparison in CI.**
2. **Independent trackers can diverge even when kernels are deterministic.** A and B must consume the same ordered source frames/fields, timestamps, calibration, model, playlist and configuration, and must initialise/reset temporal state under the same rules. Deterministic kernels reduce divergence but do not correct different inputs, packet loss or state initialisation. Do not add a synchronous cross-AZ state dependency to the frame path; instead detect divergence, fail to clean when confidence is insufficient, and prove recovery with recorded and injected-fault tests.
3. **Failover must be time-continuous.** A/B outputs and the clean bypass must carry a common source-time/frame identity. The clean path needs a calibrated delay line and the selector must switch on frame/field boundaries using embedded timecode or a validated frame counter—not when an asynchronous health check happens to fire. Also align PTS/DTS, GOP/IDR boundaries and audio/ANC delay. Test switching after decode as well as at packet level.

**Important MediaConnect distinction:** MediaConnect source *Merge* is packet recovery for binary-identical redundant streams (typically from the same encoder); it is not a frame-accurate selector for two independently rendered A/B outputs. MediaConnect source *Failover* changes source after 500 ms without data in the documented flow behaviour, which is far slower than a one-frame target. SRT sources support failover mode only (no merge), and ST 2110 JPEG XS and CDI sources do not support source failover. Do not use either as the sole implementation of the DBR A/B-to-clean switch; prove a separate broadcast-aware selector at the edge.

**Multi-region DR:** cold/warm standby in a second region is feasible but you must double capacity reservations and accept a longer path. For most operators the pragmatic tiering is: AZ redundancy for every match, region redundancy only for tier-1 competitions, with contractual clarity about what happens in a regional impairment.

### 3.9 What AWS services to use — and what not to use
| Need | Use | Do not use | Why |
| --- | --- | --- | --- |
| Contribution transport | MediaConnect for supported SRT, RIST Simple, Zixi, RTP and ST 2110 JPEG XS flows; or self-managed gateways | — | Managed transport reduces undifferentiated ops, but protocol profiles, encryption, failover semantics and quotas must be checked; use a gateway where Main-profile RIST or unsupported behaviour is required |
| The CV + compositing itself | Your own containers on EKS + EC2 GPU | MediaLive / MediaConvert / any managed transcode service | They cannot run your segmentation/warp/composite chain. This is your product; it stays yours |
| OTT packaging/distribution | MediaPackage + CloudFront | — | Different latency regime; separate pipeline |
| Catch‑up re‑renders with other ads | EKS/Batch offline GPU jobs (your renderer) | MediaConvert alone | Same compositing requirement |
| Training | SageMaker / Batch / EKS+Kueue on Spot | live node pools | Never mix training with on‑air GPUs |
| Orchestration | Step Functions + EventBridge Scheduler | cron in a pod | Fixture calendar is business logic; make it observable and retryable |
| Analytics/proof‑of‑play | S3 + Iceberg + Athena | per‑frame rows in an RDBMS | Volume |
| Secrets/keys | Secrets Manager, KMS, Pod Identity | env vars, baked creds |  |
| Chaos testing | AWS FIS — outside live windows only | during matches | Obvious |

### 3.10 Capacity & cost model (method + worked example)

**Capacity model:**

```
For each kickoff slot s, size each AZ independently:
  M_s        = concurrent matches
  K_m        = market outputs for match m
  A_m        = analysis GPUs per match replica (example: 1)
  R_m        = ceil(K_m / measured_render_outputs_per_GPU)
  G_m        = A_m + R_m   # assumes dedicated analysis and render GPUs
  G_AZ(s)    = ceil( Σ_m G_m × (1 + headroom_fraction) )
  G_region   = 2 × G_AZ(s) # A/B replicas in two AZs, before node-size rounding

If analysis and rendering share the same GPU, do not add A_m and R_m blindly: benchmark the combined workload and size from measured concurrent resource demand. If a match uses several GPUs, keep the allocation within one node unless a separately tested multi-node media fabric is explicitly designed.

Peak per AZ = max over s of G_AZ(s)  ← reserve this independently in both AZs
Mean = total weekly GPU-hours / 168  ← useful for cost reporting, not live-peak capacity
```

**Worked example [A — all inputs assumed, replace with your real calendar and benchmarks]:**

- Peak slot: 9 concurrent matches, average 12 markets each.
- Measured density: 6 market outputs per render GPU at 1080p50 low-delay (illustrative assumption; benchmark the chosen GPU and codec path).
- Analysis: 1 dedicated GPU per match replica; 2 render GPUs per match replica.
- A/B: one replica in each of two AZs; 20% headroom in each AZ.
- Node size: 8 GPUs per node (AWS lists up to 8 GPUs for G7 and G7e at the largest size; smaller sizes carry fewer, so verify per size).

```
per match replica: 1 analysis + ceil(12/6)=2 render = 3 GPUs
9 matches × 3 GPUs = 27 GPUs required in EACH AZ
27 × 1.20 = 32.4 → round up to 33 GPU capacity units required per AZ before node-size rounding
33 / 8 GPUs per node → 5 nodes per AZ (40 installed GPUs/AZ)
If reserving whole 8-GPU instances, reserve 5 instances per AZ (40 GPUs/AZ), not just 33 GPU units
2 AZs → 10 nodes, 80 installed GPUs total
```

This example assumes analysis and render GPUs are separate. If one GPU can perform both stages at the required p99.9 latency, re-run the calculation using the combined benchmark; do not apply both models at once.

Three refinements to the arithmetic above [R]:

- **Per-match rounding:** `ceil(K_m / outputs_per_GPU)` applies per match. Using the average of 12 markets hides the rounding when K varies (for example, K = 13 needs 3 render GPUs, not 2); size from the real K_m per fixture.
- **Node bin-packing:** P2 keeps a match's GPUs on one node, so also compute matches per node. At 3 GPUs per match an 8-GPU node holds floor(8/3) = 2 matches, so 9 matches need ceil(9/2) = 5 nodes per AZ, with 2 GPUs stranded on each full node. This coincides with the 5 nodes above; it will not in general.
- **Two headroom layers:** the per-GPU target utilisation (0.6, i.e. 40% headroom at p99.9, §3.7) is distinct from the fleet `headroom_fraction` (spare GPUs for failure, rounding and late fixtures). Apply both deliberately; neither covers the other.

Now the honest cost lines:

```
Live + pre-warm GPU-hours per slot ≈ 10 nodes × (2.5 h match window + 0.75 h pre-warm) = 32.5 node-hours
× node $/h [V current price]                     → dominant line in this example
+ idle ODCR capacity-hours outside the running-node hours → billed while reserved even if unused (Capacity Blocks only if the selected GPU SKU/Region is eligible)
+ egress: if both A and B replicas send all outputs continuously to the physical selector: 12 markets × 9 matches × 20 Mb/s × 2.5 h × 2 ≈ 4.86 TB → ~$437 at the illustrative $0.09/GB rate [V region, tier, destination and transfer path]. If the selector is hosted elsewhere or standby output is not continuously transported, recalculate from the actual topology.
+ MediaConnect flow-hours + per-GB [V]
+ storage, control plane, observability and EKS cluster-hour charges [V current standard/extended support rate and region]
+ offline training/QA (Spot) — usually small
```

**Levers, ranked by expected impact:**

1. **Share the analysis stage across markets (P3)** — biggest structural saving.
2. **Raise measured streams/GPU** via preset tuning, GPU-resident zero-copy compositing, and correct instance choice — direct divisor on your dominant cost.
3. **Option C (edge rendering)** — can remove per-market cloud encoding and most egress by rendering at the edge; the saving is offset by edge fleet, timing, fail-to-clean and operations cost (§3.2), so evaluate it at high K.
4. **Separate capacity reservation from billing discounts.** Use zonal ODCRs (or another currently eligible capacity product) for the live capacity floor. Compute Savings Plans can discount eligible compute spend across a broad range of EC2 usage; EC2 Instance Savings Plans generally offer a larger discount but are scoped to an instance family in a Region. Neither Savings Plan reserves launch capacity. Verify current terms and eligible usage [V].
5. **Spot for everything offline, never live.**
6. **Right-size the pre-warm window** and shut down promptly after the final whistle (but handle extra time/penalties — do not tear down at 90+15).

**Cost traps specific to this workload:**

- **Unused On-Demand Capacity Reservations are billed** — a postponed match still costs you; model postponed fixtures and idle reserved capacity explicitly.
- **Capacity Blocks for ML are limited to the instance types and Regions in the current published eligibility table; the table reviewed here does not list the G-family candidates.** Do not assume G7/G7e can be reserved with Capacity Blocks; use a suitable ODCR where offered and re-check eligibility at design freeze.
- **Cross-AZ raw media is approximately $20/h/stream** for the illustrative 1080p50 v210 assumptions in §1.3; actual transfer pricing and billing path vary.
- **NAT gateway processing on media traffic:** do not route the live media plane through NAT. Use VPC endpoints for supported AWS service traffic (for example, S3/ECR/STS/CloudWatch); design internet or private media transport separately.
- **Extended EKS support pricing on old versions** [V] — a version-upgrade procrastination tax.
- **Observability cardinality:** per-frame, per-market, per-board metrics will cost more than some of your GPUs. Aggregate at the edge, sample traces.

### 3.11 Observability & SLOs

**Three distinct layers — do not conflate them:**

| Layer | Question | Signal |
| --- | --- | --- |
| Infrastructure | Is the machine healthy? | DCGM: XID errors, ECC (SBE/DBE), row remaps, SM/NVENC/NVDEC occupancy, clocks & throttle reasons, PCIe replays, temperature; ENA: dropped/over‑limit packets (bw_in_allowance_exceeded, etc.); node/EC2 scheduled events via EventBridge |
| Pipeline | Is every frame on time and correct? | per‑stage p50/p99/p99.9 latency histograms, late/dropped/repeated frame counters, jitter‑buffer occupancy, matte age, tracker confidence, homography residual, NVENC queue depth, A/V delay delta, egress SRT retransmit rate & RTT |
| Semantic / commercial | Did the right ad actually appear in the right market for the contracted time? | sampled output frame check ("is ad X present in board slot Y"), per‑slot visibility area × duration, occlusion‑aware exposure, hash‑chained proof‑of‑play, per‑market policy compliance |

**The silent-failure problem deserves its own alarm:** if the model returns an empty matte, the output is a perfectly valid video stream showing the original ads. Every infrastructure and pipeline metric is green. The client is not receiving what they bought. **Only semantic monitoring catches this. Build it as a first-class SLO, sampled from the egress (post-encode), not from the compositor's internal state.**

**Candidate SLOs [A/R] — agree the measurement contract before using these as commitments:**

- **Visible artefacts:** target fewer than one adjudicated visible-artefact incident per match per market. Define an incident as a contiguous run of bad frames/fields, specify the severity threshold, and measure at the decoded egress; do not equate this directly with a simple per-frame availability percentage.
- **Time-to-fail-to-clean:** target ≤ 1 progressive frame or interlaced field, with no programme-time jump. This is a stringent design target that must be demonstrated with the physical selector and clean delay line.
- **Ad exposure delivered vs contracted:** set the numeric threshold as **TBD by contract** after agreeing the visibility/occlusion methodology, denominator, sampling rules, tolerances and make-good treatment. Do not carry an undefined percentage placeholder into a signed SLO.
- **A/V delay divergence:** candidate limit ≤ 1 field; validate against the broadcaster's acceptance test and audio format.
- **Proof-of-play report availability:** T + 2 h is a candidate operational target, not a universal broadcast standard; agree retention, reconciliation and dispute windows with clients.

### 3.12 Security, rights, compliance

- **Contribution feeds are high-value, piracy-attractive:** use protocol encryption where supported (for example, configured SRT/Zixi encryption), private paths where practical, IP allowlists, no unnecessary public media endpoints and per-market entitlements. AWS Elemental MediaConnect currently documents that it does **not** support encryption for RIST sources or outputs; protect that leg with a private network or an independently qualified encrypted gateway, and verify the complete path end to end.
- **Forensic watermarking per market** for leak attribution — costs compute; decide per competition tier.
- **IAM least privilege** via EKS Pod Identity, KMS for assets/logs, private EKS endpoint, VPC endpoints, no credentials in images.
- **Data residency:** EU competitions may require EU processing/logs; region choice is then a legal constraint, not a latency optimisation.
- **GDPR/privacy:** incidental crowd images in recorded material may be personal data. Apply data minimisation, retention/access controls and a documented privacy assessment; complete a DPIA where the proposed processing is likely to result in high risk or otherwise required. Avoid treating a DPIA as automatically required for every form of audience analytics.
- **Media supply chain audits:** TPN / ISO 27001 / SOC 2 expectations from broadcasters; league security questionnaires.
- **Ad content compliance per territory** (gambling, alcohol, political, competitor exclusivity, league vs club sponsor hierarchy) enforced by a policy engine with an audit trail, and a **kill switch per market/slot usable within seconds**.

---

## 4. CHALLENGE REGISTER

**Severity:** 🔴 can take you off air / breach contract · 🟠 degrades quality or margin · 🟡 operational friction.

### 4.A inherent challenges

| ID | Challenge | Mechanism | Sev | Mitigation | Residual |
| --- | --- | --- | --- | --- | --- |
| INH‑1 | Player/official occlusion of boards | Segmentation must place ads behind legs, bodies, ball, flags, staff | 🔴 | Instance segmentation + temporal matting + explicit occlusion reasoning; boundary‑F‑score metric | Fast crossings, thin limbs, motion blur |
| INH‑2 | Matte flicker / temporal instability | Per‑frame independent inference jitter at sub‑pixel edges | 🔴 | Temporal propagation/flow, hysteresis, edge‑stability loss term, temporal‑consistency QC gate | Trade‑off: smoothing adds lag on fast pans |
| INH‑3 | Camera cuts and re‑identification | New shot ⇒ new geometry; homography cache miss | 🔴 | Cut detector + camera fingerprint classifier + per‑camera calibration cache; safe fallback (hold last matte only if geometry unchanged, else bypass) | Unknown/new camera positions |
| INH‑4 | Tight close‑ups: few pitch lines | Homography from lines becomes ill‑conditioned | 🟠 | Board‑edge tracking fallback, lens/zoom encoder metadata if available, previous‑frame propagation with confidence decay | Extreme zoom + defocus |
| INH‑5 | Motion blur & defocus mismatch | Ad rendered sharp on a blurred board looks fake | 🟠 | Estimate per‑region blur kernel from image; apply matching defocus + directional motion blur; grain/noise matching | Estimation error visible to trained eyes |
| INH‑6 | LED board artefacts: refresh beating, moiré, banding | Board PWM/refresh vs camera shutter; rolling shutter | 🟠 | Shutter/refresh awareness, banding detection, temporal filtering; coordinate with board vendor | Physics‑limited |
| INH‑7 | Weather/lighting: rain, fog, snow, low sun, shadow edges, pyro/confetti | Segmentation domain shift | 🟠 | Augmented training, per‑stadium adaptation, confidence‑triggered bypass | Rare extremes |
| INH‑8 | Replays, slow‑mo, DVE, split screens, wipes | Two geometries in one frame; speed ramps break trackers | 🔴 | Replay/detect classifier; per‑region geometry; handle 0.25×–3× and reverse; if ISO‑based, cover EVS feeds | Complex graphics packages |
| INH‑9 | On‑screen graphics/scorebug/VAR lines | Must not be overpainted or misclassified | 🔴 | Graphics detection + protected regions; z‑order rules agreed with host broadcaster | New graphics each season |
| INH‑10 | Interlace (1080i50) field accuracy | Fields 20 ms apart, different geometry | 🔴 | Field‑level processing or motion‑aware field pairing; avoid naive deinterleave/reinterleave | Cost doubles inference rate |
| INH‑11 | Colour/brightness realism | Ad asset sRGB vs BT.709/2020 legal range; LED board response | 🟠 | Colour pipeline with legal‑range handling, per‑board luminance/white‑balance match, HLG/709 conversions | Per‑board calibration effort |
| INH‑12 | A/V and ANC delay alignment | Video path adds delay; audio/ANC must match exactly | 🔴 | Single delay line for video+audio+ANC; ANC passthrough or re‑insert; measure A/V delta continuously | Dolby metadata passthrough fragility |
| INH‑13 | Clock drift ingest↔egress | 100 ppm ⇒ 0.72 s over 2 h | 🔴 | Slave egress clock to source timestamps; audited drop/repeat policy; audio ASRC | Requires disciplined implementation |
| INH‑14 | Silent failure (no replacement) | Empty matte ⇒ valid video, wrong content | 🔴 | Semantic egress monitoring + commercial SLO + auto‑bypass/alert | Sampling gap |
| INH‑15 | Ad edge bleed / partial original visible | Matte under‑covers board boundary | 🟠 | Conservative geometry dilation with occlusion respect; boundary QC | Sub‑pixel cases |
| INH‑16 | Double replacement / downstream conflict | Another vendor or the physical board system also replaces | 🟠 | Contractual + technical coordination, slot ownership registry | Multi‑vendor events |
| INH‑17 | Proof‑of‑play disputes | Exposure measurement contested by agency/client | 🟠 | Frame‑accurate, occlusion‑aware, timecode‑referenced, hash‑chained logs; third‑party reconciliation | Methodology disagreements |
| INH‑18 | Territorial ad legality & sponsor conflicts | Gambling/alcohol bans, league vs club hierarchy | 🔴 | Policy engine per market/slot/category, pre‑match sign‑off, per‑market kill switch | Late regulatory changes |
| INH‑19 | Last‑minute asset changes / sponsor withdrawal | Creative pulled at T‑10 min | 🟠 | Hot‑reloadable asset/playlist without pipeline restart; validation gate; versioned rollback | Human error under time pressure |
| INH‑20 | Fixture volatility | Delays, extra time, penalties, abandonment, schedule moves | 🟠 | Event‑driven orchestration with extend/hold logic; capacity reservations with buffer; never tear down at 90+15 | Weather abandonments |
| INH‑21 | Simultaneous‑kickoff stampede | 5–15 pipelines start within the same minute; engine load + calibration storm | 🔴 | Staggered pre‑warm, capacity reserved, engines pre‑built, admission control | Peak sizing cost |
| INH‑22 | Model drift season‑to‑season | New boards, sponsors, kits, pitch, stadium works, camera plan | 🟠 | Per‑event pre‑flight validation, active learning on failures, regression suite per stadium | Continuous labelling cost |
| INH‑23 | Access rights to feeds | You cannot process what you don't receive; ISO vs PGM; host broadcaster cooperation | 🔴 | Contractual feed access, ISO/tally metadata where possible, PGM‑only fallback design | Out of your control |
| INH‑24 | Latency vs second screen / betting sync | Different markets on different delays ⇒ spoilers, sync complaints | 🟠 | Publish per‑market delay, align where contractually required | Cannot fully equalise physics |
| INH‑25 | Human ops at 21:00 on a Saturday | Match‑critical changes made by tired humans | 🟠 | Match‑day change freeze, four‑eyes for live actions, runbooks, rehearsal | Irreducible |

### 4.B current-architecture challenges

| ID | Challenge | Mechanism | Sev | Mitigation |
| --- | --- | --- | --- | --- |
| DC‑1 | Peak‑sized capex idle most of the week | Fixture calendar burstiness (§2.8) | 🟠 | Utilisation‑aware scheduling; sell idle to offline jobs; this is the core cloud argument |
| DC‑2 | Cannot absorb a new client/competition quickly | GPU procurement + rack + power lead time (weeks–months) | 🔴 | Forward capacity plan tied to sales pipeline; cloud burst as overflow |
| DC‑3 | Single‑site concentration risk | Power, cooling, fire, fibre cut, upstream ISP | 🔴 | Second site or cloud DR; diverse fibre entries; teleport diversity |
| DC‑4 | Power/cooling density ceiling | GPU racks exceed designed kW/rack | 🟠 | Containment, rear‑door/DLC, spread racks; derate |
| DC‑5 | Thermal throttling ⇒ inference jitter | Dense GPUs, dust, fan failure, hot aisle recirculation | 🟠 | DCGM clock/throttle alerting, thermal SLO, capacity headroom |
| DC‑6 | Self‑managed control plane burden | etcd HA/defrag/backup, cert rotation, upgrades, admission webhook blast radius | 🟠 | Managed EKS in target; until then: strict upgrade runbooks, etcd drills |
| DC‑7 | CNI/CSI/multicast/PTP complexity | Media plane needs SR‑IOV, multicast, jumbo, HW timestamps outside normal CNI | 🟠 | Multus + dedicated media VRF; documented NIC/PTP config; config drift detection |
| DC‑8 | ST 2110 microburst loss | Uncompressed senders burst; switch buffers overflow; ST 2110‑21 compliance | 🔴 | Headroom, traffic shaping, receiver buffer tuning, per‑flow telemetry |
| DC‑9 | PTP grandmaster / genlock failover | GNSS loss, GM switch, boundary clock misconfig | 🔴 | Dual GM, GNSS holdover quality, continuous PTP monitoring |
| DC‑10 | GPU driver/firmware/kernel upgrade vs uptime | Driver ↔ CUDA ↔ TensorRT ↔ kernel coupling; no live migration | 🟠 | Blue/green nodes, drain only outside matches, pinned versions, staged rollout |
| DC‑11 | Noisy neighbours on shared GPUs | Time‑slicing/MPS without hard isolation; OOM/XID kills neighbours | 🔴 | Exclusive GPU allocation per live match replica where possible; if several GPUs are needed, allocate them within the same node. Use sharing only for offline/QA unless an isolation profile is qualified |
| DC‑12 | CFS throttling / NUMA crossing | Container CPU limits + scheduler ⇒ ~100 ms stalls | 🔴 | Guaranteed QoS, static CPU pinning, single‑NUMA topology, hugepages |
| DC‑13 | Hardware failures & RMA cycle | GPU/NIC/PSU/disk failures; spares logistics | 🟠 | N+1, on‑site spares, vendor SLAs, burn‑in |
| DC‑14 | Hardware‑bound software licences | USB dongles (CodeMeter/Sentinel), MAC‑bound codec SDKs | 🟠 | Licence servers, vendor network licences; inventory & renewal tracking |
| DC‑15 | Satellite/fibre contracts & teleport cost | Long‑term leases, occasional‑use slots, uplink booking | 🟠 | Renegotiate to cloud‑friendly contribution; DX to teleport |
| DC‑16 | 24/7 broadcast + IT skill mix in one team | Rare combination; on‑call fatigue; key‑person risk | 🟠 | Documented runbooks, cross‑training, follow‑the‑sun if possible |
| DC‑17 | Archive growth & data gravity | Multi‑season recordings (PB scale) | 🟠 | Tiering, retention policy, lifecycle to cold storage |
| DC‑18 | Security patching without off‑air windows | Live calendar leaves few safe windows | 🟠 | Match‑aware maintenance calendar; international breaks as patch windows |
| DC‑19 | Capacity fragmentation | Free GPUs exist but not with the right NUMA/NIC/topology | 🟡 | Topology‑aware scheduling, defragmentation windows |
| DC‑20 | Refresh/depreciation cliff | Whole fleet ages together; big periodic capex | 🟠 | Staggered refresh; cloud shifts this to opex (at a price) |

### 4.C AWS/EKS challenges — capacity, compute, network and operations

| ID | Challenge | Mechanism | Sev | Mitigation |
| --- | --- | --- | --- | --- |
| AWS‑1 | Insufficient capacity at kickoff | GPU demand is correlated across the whole industry (Sat 15:00, midweek evenings); on‑demand launches can fail | 🔴 | ODCRs for exact G-family SKU/AZ where available; do not assume Capacity Blocks for G instances; multi-AZ reservations; pre-warm T-45; admission control with explicit "cannot accept this match" state |
| AWS‑2 | Reserved capacity costs when unused | ODCR billed whether used or not; Capacity Blocks fixed term | 🟠 | Reserve the floor, burst the tail; contractual buffer for postponements; portfolio across competitions |
| AWS‑3 | GPU family/AZ availability skew | Not all G families in all regions/AZs; capacity differs per AZ | 🟠 | Verify per‑AZ offer; abstract via NodePool with several types; keep determinism pairs matched |
| AWS‑4 | Service quotas block the season | G/VT vCPU quota, ENIs, flows, DX ports | 🔴 | Raise quotas weeks ahead; quota monitoring + alarms; load test at 2× peak |
| AWS‑5 | GPU isolation differs by SKU | A10G/L4/L40S have no MIG; Blackwell RTX PRO 4500/6000 support MIG at GPU level but EC2/EKS enablement and codec-engine partitioning must be checked | 🟠 | Prefer exclusive GPUs or tested MIG profiles for live; never mistake DRA or time-slicing for hardware isolation; validate NVENC/NVDEC visibility per profile |
| AWS‑6 | Instance maintenance / retirement mid‑match | No live migration for GPU instances | 🔴 | AWS Health/EventBridge scheduled‑event feed into orchestrator; match‑aware drain locks; N+1 in a second AZ; pre‑emptive replacement before windows |
| AWS‑7 | GPU XID / ECC / fallen‑off‑bus | Hardware faults | 🔴 | DCGM XID alerting → immediate fail‑to‑clean + node cordon + replace; A/B independence |
| AWS‑8 | Cold start ≫ kickoff tolerance | Node boot + 10 GB image + engine build | 🔴 | Pre‑built engines, SOCI/AMI‑baked images, warm nodes, balloon pods, scheduled pre‑warm |
| AWS‑9 | Spot reclamation | 2‑minute warning, correlated reclaim at peak | 🔴 | Never live on Spot; Spot only offline with checkpointing |
| AWS‑10 | Karpenter/CA consolidates a live node | Cost optimiser sees "underutilised" media node | 🔴 | do-not-disrupt, NodePool disruption budgets with match‑window crons, PDBs, descheduler off during windows |
| AWS‑11 | Liveness probe restarts an on‑air pipeline | Transient stall trips probe | 🔴 | Probe philosophy: degrade→bypass, not kill; readiness for admission only |
| AWS‑12 | A/B output divergence breaks seamless failover | Different GPU/engine/precision, source-frame order or tracker state ⇒ different mattes | 🔴 | Pin hardware/software/configuration; align source timestamps and frame identity; deterministic kernels; continuous A/B divergence checks and CI pixel/mask comparison |

**Network, timing, transport**

| ID | Challenge | Mechanism | Sev | Mitigation |
| --- | --- | --- | --- | --- |
| AWS‑13 | No genlock / no broadcast PTP plant | EC2 offers NTP and PHC on supported precision-time placements, not an ST 2059-2 grandmaster/tri-level chain | 🔴 | Timestamp-based media-clock recovery; keep timing island on prem; accept frame/field drop-repeat policy only with audit |
| AWS‑14 | Limited multicast support | Transit Gateway supports multicast between attached VPC subnets, but documented throughput is limited (1 Gb/s per flow, not adjustable; 20 Gb/s aggregate per AZ), multicast is not routed over Direct Connect, and it is not an ST 2110 fabric | 🟠 | Prefer unicast SRT/RIST/Zixi for contribution; use TGW multicast only after an explicit quota/performance test; keep 2022-7/broadcast merge at the physical edge |
| AWS‑15 | Per-flow bandwidth ceilings | Instance-to-instance single-flow limits depend on placement group and ENA capability; service/public paths differ | 🟠 | Compressed transport; measure packet-size and sustained throughput; use cluster placement or ENA Express only where eligible and justified |
| AWS‑16 | Cross-AZ raw media cost/latency | Approximately $20/h per 1080p50 v210 stream under the illustrative transfer assumptions in §1.3 | 🔴 | AZ-local replicas; keep raw-frame traffic node-local and do not replicate uncompressed frames across AZs |
| AWS‑17 | "Up to X Gbps" burst networking | Baseline ≠ burst; allowance‑exceeded drops | 🟠 | Choose sustained‑bandwidth sizes; monitor ENA allowance metrics; shape egress |
| AWS‑18 | Internet path variance on contribution | Last mile at stadium is often the weak link | 🔴 | Diverse paths/bonding, DX where practical, optional Global Accelerator UDP ingress after test, and per-venue ARQ buffers sized against measured loss/jitter |
| AWS‑19 | ARQ buffer ↔ latency trade‑off | SRT latency must be set from measured RTT and jitter; 3–4 × RTT is a heuristic, not a protocol guarantee | 🟠 | Measure RTT/loss per venue; per‑venue latency profiles; prefer XS/DX where latency‑critical |
| AWS‑20 | Satellite contribution dominates budget | ~240 ms+ one way | 🟠 | Prefer fibre; if satellite is mandatory, process near the receive point |
| AWS‑21 | Intercontinental hairpinning | stadium→region A→continent B | 🟠 | Source‑local processing + fan‑out; Option C edge rendering |
| AWS‑22 | NAT/overlay/mesh in the frame path | Extra hop, extra jitter, extra cost | 🔴 | Avoid NAT/mesh in media path; use host networking or a qualified secondary-interface design; VPC endpoints for control/assets; no sidecars on media |
| AWS‑23 | NLB flow re‑hash on reconnect | UDP flow → different target, state lost | 🟠 | Pinned targets/EIPs for media; explicit reconnect semantics; state externalised for control only |
| AWS‑24 | ANC/timecode loss through IP/cloud chain | Many stacks drop ANC | 🔴 | Explicit ANC passthrough/re‑insert; own SEI reference; verify end‑to‑end per market |
| AWS‑25 | Managed-service codec/protocol gaps | MediaConnect supports ST 2110 JPEG XS, but this is not uncompressed ST 2110-20; RIST is Simple profile only and unencrypted in MediaConnect | 🟠 | Confirm the exact source/output protocol, profile, codec, bit depth, flow limits, Region and failover behaviour; retain a self-managed gateway fallback |

**Platform & operations**

| ID | Challenge | Mechanism | Sev | Mitigation |
| --- | --- | --- | --- | --- |
| AWS‑26 | EKS/K8s version treadmill | Standard support window per minor version, then paid extended; no downgrade | 🟠 | Blue/green clusters per season; upgrade in international breaks; version‑skew policy |
| AWS‑27 | Driver/CUDA/TensorRT/AMI matrix drift | GPU Operator + AMI + your image must agree | 🔴 | Pin everything; CI matrix build; canary node; rollback plan |
| AWS‑28 | Engine per (arch × shape × precision) explosion | TensorRT engines are not portable | 🟠 | Build farm; cache keyed by hash; restrict supported shapes; dynamic‑shape profiles with measured cost |
| AWS‑29 | Observability cardinality/cost | Per‑frame × per‑market × per‑board metrics | 🟠 | Edge aggregation, histograms not per‑frame events, sampled traces, separate commercial pipeline |
| AWS‑30 | Control‑plane dependency for edge/hybrid | Outposts/Local Zones/hybrid nodes depend on parent region connectivity | 🟠 | Running pods survive partition, but no rescheduling; local fallback logic; explicit degraded mode |
| AWS‑31 | Cloud licence terms for codecs/SDKs | Hardware‑bound licences don't work on EC2; per‑GPU AI software licences | 🟠 | Vendor cloud/network licences negotiated before migration; legal review of codec patent licensing |
| AWS‑32 | Egress/transfer cost surprises | Per‑GB internet, DX, cross‑region, NAT | 🟠 | Cost model per match; DX for broadcaster handoff; Option C; budget alarms per competition |
| AWS‑33 | Multi‑account/region config drift | Same pipeline must behave identically everywhere | 🟠 | GitOps + Terraform/Crossplane, conformance tests, immutable images, no manual changes |
| AWS‑34 | Skills gap: broadcast engineers ↔ cloud SRE | Different mental models (genlock vs NTP, deterministic vs elastic) | 🟠 | Joint runbooks, paired on‑call, cross‑training, explicit "cloud cannot do X" documentation |
| AWS‑35 | Loss of physical control during incidents | You cannot walk to the rack | 🟠 | Contractual escalation paths, AWS support tier, pre‑agreed incident comms with broadcasters |
| AWS‑36 | Data gravity / exit cost | PB archive + egress fees make reversal expensive | 🟠 | Exit plan up front; keep archive tiering portable; avoid proprietary‑only data formats |
| AWS‑37 | Region impairment | Rare but real multi‑AZ regional events | 🔴 | Tiered DR: AZ for all, warm region for tier‑1; contractual definition of degraded service |
| AWS‑38 | Noisy multi-tenant network tail latency | Shared cloud fabric vs dedicated leaf/spine | 🟠 | Measure p99.9 continuously; evaluate cluster placement/ENA Express only for eligible instance pairs; size buffers on measured tails |
| AWS‑39 | Security/compliance re‑certification | Broadcasters/leagues audit the new environment | 🟠 | TPN/ISO/SOC2 alignment, evidence automation, residency controls, encryption everywhere |
| AWS‑40 | Cost of dual‑running | Two full stacks during migration | 🟠 | Time‑box shadow mode; per‑competition cutover; explicit exit criteria |

### 4.D Migration-specific challenges
| ID | Challenge | Mechanism | Sev | Mitigation |
| --- | --- | --- | --- | --- |
| MIG‑1 | Output parity is not "looks similar" | Sub-pixel matte differences, colour shifts, codec profile/chroma changes and different encoder behaviour | 🔴 | Frame/field-aligned shadow comparison: mask IoU, boundary F-score, temporal stability, colour/legal-range checks, decoded pixel metrics and human blind review; use VMAF only as a supplementary perceptual metric, not as a substitute for mask/edge validation |
| MIG‑2 | Latency parity impossible to fake | Cloud path is physically longer than a rack in the same room | 🔴 | Renegotiate delay budgets with broadcasters before cutover; re‑align all feeds; publish per‑market delay |
| MIG‑3 | Timing/ANC integration at the new boundary | Cloud returns IP; broadcaster expects genlocked SDI | 🔴 | Keep physical handoff island; frame sync + ANC re‑insert + delay match; test with each broadcaster |
| MIG‑4 | Failover semantics change | 2022‑7 hardware merge vs cloud failover flow | 🔴 | Prove frame‑accurate, time‑continuous switching in rehearsal; inject faults deliberately (off air) |
| MIG‑5 | Fixture calendar leaves no test windows | Live every weekend + midweek | 🟠 | Synthetic replay harness with ground truth; rehearse on recorded matches at full rate; use international breaks and lower‑tier fixtures for canaries |
| MIG‑6 | Big‑bang risk | Cutting all markets at once | 🔴 | Strangler pattern: shadow → one low‑stakes market → one competition → tier‑1; DC remains fallback for a full season |
| MIG‑7 | Archive migration | PB over finite bandwidth | 🟠 | DX + parallel transfer, physical transfer appliances where appropriate [V current portfolio]; migrate hot data first, leave cold in DC until contract end |
| MIG‑8 | Licence renegotiation blockers | Vendor refuses cloud licensing | 🔴 | Identify every dongle/hardware‑bound licence in week 1; replace codec/SDK if needed (re‑validation cost) |
| MIG‑9 | Operational muscle memory | Old runbooks reference racks and SDI routers | 🟠 | Rewrite runbooks per failure mode; game days; joint broadcast+cloud incident drills |
| MIG‑10 | Contractual/SLA renegotiation | Delay, DR tier, measurement methodology, liability | 🔴 | Legal workstream parallel to engineering; do not cut over before SLAs match reality |
| MIG‑11 | Hidden dependency discovery | Calibration tools tied to a workstation; a cron on one node; a licence server in a rack | 🟠 | Dependency inventory + traffic capture before design freeze; assume you will find more |
| MIG‑12 | Reversibility | Cloud‑only features creep into the product | 🟠 | Keep the pipeline cloud‑agnostic (containers, S3‑compatible API, no proprietary live‑path dependency) until DC is formally retired |

---

## 5. Migration plan (phase gates, not dates)
| Phase | Content | Exit gate (must be measurable) |
| --- | --- | --- |
| 0. Inventory & measurement | Full dependency map; per‑stage GPU/encode profiling on candidate EC2 types; per‑venue RTT/loss; licence audit; quota requests; latency budget agreement in principle | Benchmarks published; capacity model built from the real calendar; licences cleared or replacement identified |
| 1. Landing zone + platform | Accounts, VPC/DX, EKS, node pools, GPU stack, GitOps, observability, security baseline | Cold‑start time measured ≤ target; capacity reservation proven in the target AZs; 2× peak load test passes |
| 2. Offline parity | Recorded matches replayed through cloud pipeline at full rate; frame‑aligned comparison vs DC output | Matte IoU / boundary F / temporal stability thresholds met on a representative corpus (VMAF reported as a supplementary metric only, per MIG-1) (multiple stadiums, weather, interlace, replays, graphics) |
| 3. Shadow live | Real fixtures processed in cloud in parallel; outputs not delivered; semantic monitoring exercised | ≥ N matches (N agreed before entering the phase) with zero unexplained divergences; failover injections produce ≤ 1 frame impact and no time jump |
| 4. Canary market | One low‑stakes territory delivered from cloud, DC as instant fallback | Contractual exposure delivered; broadcaster signs off timing/ANC; incident‑free for M matches (M agreed before entering the phase) |
| 5. Per‑competition rollout | Competition by competition, market by market; DC kept warm | Two full matchdays at peak concurrency without capacity failure |
| 6. Tier‑1 + DR | Tier‑1 competitions; multi‑AZ everywhere; region DR for tier‑1 | Documented, rehearsed regional failover; SLA renegotiated and signed |
| 7. DC retirement / repurpose | DC becomes the physical/timing island + edge renderers (Option C) + offline capacity | One full season with cloud primary and no DC fallback invocation |

**Non-negotiable rule for every phase:** the physical fail-to-clean path with matched delay exists and is tested before any market is delivered from cloud.

---

## 6. What should not move to AWS (and why — logical, not sentimental)

| Function | Why it stays physical |
| --- | --- |
| SDI/2110 plant, genlock, PTP GM, frame syncs | AWS provides no equivalent; broadcast integration requires them |
| SMPTE 2022‑7 seamless merge / bypass relay | Timing‑critical, sub‑frame, must survive cloud loss |
| Satellite reception (dishes, IRDs) | Physics |
| LED perimeter board control, IR/key overlay systems if used | In‑venue hardware, camera‑synchronised |
| Stadium big screen / in‑venue feed | Must never show virtual ads |
| Hardware‑bound licence dongles (until renegotiated) | Cannot attach USB to EC2 |
| Final broadcaster playout / MCR handoff | Not yours, and genlocked to their plant |
| Last‑resort clean fallback | Must work when the cloud path does not |

Everything else — CV analysis, compositing, encoding, ad scheduling, proof-of-play, training, archive — is a legitimate cloud candidate.

---

## 7. Open questions & the measurement plan (what I need from you to make this quantitative)

### Business/format inputs

1. Exact fixture calendar for one season, with concurrency per kickoff slot, and markets per match per competition.
2. Delivery formats per market (1080i50 vs p50 vs UHD/HLG), codecs, bitrates, and required added latency per market.
3. Do you receive PGM only, or ISOs + tally? Do you process replays from PGM or from EVS?
4. Contractual exposure measurement methodology and who arbitrates disputes.
5. Per-market regulatory constraints and approval workflows.

### Benchmarks to run (these determine the entire cost model)

6. Per-stage latency and GPU cost at required formats on G7 (RTX PRO 4500 Blackwell), G7e (RTX PRO 6000 Blackwell), G6 (L4), G5 (A10G) and G6e (L40S), where available — including codec-specific decode/encode, NVENC/NVDEC occupancy, memory bandwidth, PCIe/P2P transfers and p99.9 latency.
7. Measured market outputs per GPU at your exact encode preset, with ≥40% headroom, at p99.9.
8. Cold-start time end-to-end (node → first correct frame) with and without SOCI/AMI baking.
9. Determinism test: A/B on identical GPU model/engine — bit equality or measured PSNR of mattes.
10. Per-venue contribution RTT/loss distribution over a season; derived SRT latency per venue.
11. Failover impact test: injected pod kill, GPU XID simulation, AZ isolation — measured in frames, not seconds.
12. Semantic monitor efficacy: inject an empty matte and confirm detection latency.

### AWS specifics to verify [V] before design freeze

13. GPU instance families available per target region and AZ; ODCR / Capacity Block eligibility per family.
14. Current MediaConnect protocol/codec support and limits (SRT, RIST Simple only, Zixi Push/Pull, ST 2110 JPEG XS; encryption and source failover semantics), quotas and pricing dimensions.
15. Which instance types/AZs support the precision-time placement group and PHC; measure achieved clock offset and packet timestamp accuracy in the actual VPC path.
16. Current single-flow bandwidth limits and placement-group effects for your chosen instances.
17. Local Zones / Outposts GPU families in your metros; service availability there.
18. EKS Hybrid Nodes OS/GPU/add-on support scope and partition behaviour.
19. Current EKS support windows (standard/extended) and pricing; Kubernetes/DRA version and NVIDIA DRA driver maturity.
20. NVIDIA Video Encode/Decode Support Matrix for exact NVENC/NVDEC engine counts and H.264/HEVC 4:2:2 10-bit support per GPU/MIG profile; confirm driver and SDK support on EC2.
21. Current egress/DX/cross-AZ/NAT pricing in your regions; Savings Plan terms.
22. Codec/SDK/AI-software licensing terms on cloud (per-GPU, per-stream, per-instance).

---

## 8. One-page summary of the argument

1. **The workload separates into shared match analysis and per-market rendering/encoding.** Share the market-independent CV stages where the input and policy model allow it; render and encode each required territory output.
2. **Treat a match as one scheduling/failure unit and keep its live frame path node-local.** Use one GPU when measured capacity permits; otherwise use multiple GPUs on one node with measured GPU-local transfers. Avoid cross-AZ raw-frame transport.
3. **Broadcast timing has no cloud-equivalent plant.** AWS offers NTP and, on supported EC2 placements, PHC; Transit Gateway offers limited multicast, not an ST 2110/genlock environment. Keep physical sync, SDI/IP conversion, frame/field switching and clean bypass at the edge.
4. **Cold start exceeds kickoff tolerance.** Reserve and pre-warm capacity against the fixture calendar; do not rely on reactive autoscaling. Block Karpenter consolidation, routine drains and restart-triggering liveness probes during matches.
5. **A 2-hour 50-unit/s match contains 360,000 progressive frames or interlaced fields.** Design independent A/B replicas and a delay-matched clean path with frame/field-aware switching; restart-based recovery is not seamless.
6. **The GPU shortlist must satisfy the broadcast codec.** G7/G7e Blackwell are now relevant candidates for 4:2:2 H.264/HEVC encode/decode; G5/G6/G6e remain candidates only when their codec support meets the source/output requirements.
7. **The cloud business case is peak-to-mean utilisation and site resilience, not an assumed lower unit price.** Model idle reservations, pre-warm, dual AZ capacity, egress, managed media transport and dual-running.
8. **The most dangerous failure is silent:** a valid stream with the wrong ads. Use independent egress semantic checks and commercially agreed proof-of-play evidence.
9. **Option C (central analysis plus edge rendering)** may reduce cloud per-market encoding and egress when edge placement is practical, but needs a defined picture/matte interface, a renderer fleet and edge fail-to-clean.

---

## 9. Inputs required to turn this reference design into a priced deployment

The next design iteration requires the real fixture calendar, concurrent match counts, markets per match, source/output formats, contractual latency and reliability targets, plus the benchmark results listed in §7. With those inputs, the capacity and cost models can be recalculated, instance families selected by role, and the node-pool/pod specifications and failover state machine completed.

Document conventions
Term	Meaning
DBR	Digital Board Replacement (virtual replacement of perimeter LED board content)
[A] / [R] / [V]	Assumption / Recommendation / Verify at design freeze (§0 markers)
INH / DC / AWS / MIG (+ number)	Challenge-register ID prefixes: inherent / current data centre / AWS-EKS / migration (§4)
K	Territory (market) outputs per match
N	Concurrent matches (§0.1); in §5 gates, N and M are separate placeholders for match counts
M_s, K_m, A_m, R_m, G_m	Capacity-model variables: concurrent matches in slot s, outputs of match m, analysis GPUs, render GPUs, total GPUs per replica (§3.10)
A/B	The two redundant processing replicas
1+1, N+1	One active plus one standby; N units plus one spare
Broadcast and venue
Term	Meaning
OB	Outside Broadcast (production compound or truck at the stadium)
PGM	Programme output (the on-air "world" feed from the vision mixer)
ISO	Isolated camera feed (not the ISO 27001 standard)
CCU	Camera Control Unit
MCR	Master Control Room (the broadcaster's)
IBC	International Broadcast Centre
EVS	Vendor name, used generically for live replay servers
DVE	Digital Video Effects (split screens, squeeze-backs, wipes)
VAR	Video Assistant Referee
IR	Infrared (marker-based board systems)
CV	Computer Vision
IRD	Integrated Receiver/Decoder
LNB / BUC	Low-Noise Block downconverter / Block Upconverter (satellite dish electronics)
DVB-S2	Digital Video Broadcasting, Satellite, second generation
DWDM	Dense Wavelength Division Multiplexing
OTT	Over-the-Top (internet delivery to viewers)
Video, timing and transport
Term	Meaning
SDI	Serial Digital Interface (HD-SDI 1.5G, 3G-SDI, 12G-SDI)
SMPTE / ST	Society of Motion Picture and Television Engineers / its standards
ST 2110	Professional media over IP (-20 uncompressed video, -21 sender timing, -22 compressed video, e.g. JPEG XS)
ST 2059-2	PTP profile for broadcast timing
ST 2022-6 / -7	SDI over IP / seamless packet-level protection switching
PTP / NTP	Precision Time Protocol / Network Time Protocol
PHC	PTP Hardware Clock
GM	Grandmaster (PTP master clock)
GNSS	Global Navigation Satellite System
SRT	Secure Reliable Transport
RIST	Reliable Internet Stream Transport (Simple / Main profiles)
RTP	Real-time Transport Protocol
Zixi	Proprietary transport protocol (product name)
CDI	Cloud Digital Interface (AWS uncompressed-video transport)
J2K / XS	JPEG 2000 / JPEG XS (compressed mezzanine codecs)
ARQ / FEC	Automatic Repeat reQuest / Forward Error Correction
HLS / DASH / CMAF	HTTP Live Streaming / Dynamic Adaptive Streaming over HTTP / Common Media Application Format
1080i50, 1080p50, 2160p50	Resolution, interlaced (i) or progressive (p), 50 fields or frames per second
4:2:2 / 4:2:0	Chroma subsampling schemes
NV12 / P010 / v210	Pixel formats: 8-bit 4:2:0 / 10-bit 4:2:0 / packed 10-bit 4:2:2
BT.709 / BT.2020	ITU-R colour standards (HD / UHD)
HLG / UHD / HDR	Hybrid Log-Gamma / Ultra High Definition / High Dynamic Range
ANC	Ancillary data (carried in the SDI/IP stream)
TC / ATC	Timecode / Ancillary Time Code (carrying LTC or VITC)
LTC / VITC	Linear Timecode / Vertical Interval Timecode
AFD	Active Format Description
SCTE-104 / -35	Ad-insertion signalling standards (on SDI / in transport streams)
SEI	Supplemental Enhancement Information (H.264/HEVC metadata NAL units)
PTS / DTS	Presentation / Decoding Time Stamp
GOP / IDR	Group of Pictures / Instantaneous Decoder Refresh frame
ASRC	Asynchronous Sample-Rate Conversion (audio)
ppm	Parts per million (clock drift)
PWM	Pulse-Width Modulation (LED board dimming)
GPU and ML
Term	Meaning
NVENC / NVDEC	NVIDIA fixed-function hardware video encoder / decoder
CUDA / cuDNN	NVIDIA compute platform / deep-neural-network library
TRT / TensorRT	NVIDIA inference optimiser; produces per-GPU "engines"
ONNX	Open Neural Network Exchange (model format)
TF32 / FP32	TensorFloat-32 / 32-bit floating point
SM	Streaming Multiprocessor
MIG	Multi-Instance GPU (hardware partitioning); not the migration ID prefix
MPS	Multi-Process Service (GPU sharing)
DCGM	NVIDIA Data Center GPU Manager (telemetry)
XID	NVIDIA driver error-event code
ECC / SBE / DBE	Error-Correcting Code / Single- / Double-Bit Error
P2P	Peer-to-peer (GPU-to-GPU transfers)
PCIe, NVMe	PCI Express, Non-Volatile Memory Express
IoU	Intersection over Union (mask overlap metric)
PSNR	Peak Signal-to-Noise Ratio
VMAF	Video Multimethod Assessment Fusion (perceptual quality metric)
G5/G6/G6e/G7/G7e	EC2 GPU instance families (A10G / L4 / L40S / RTX PRO 4500 / RTX PRO 6000)
Inf2 / Trn1	AWS Inferentia2 / Trainium1 (Neuron accelerators)
SKU	Stock-keeping unit (a specific instance or GPU model)
AWS
Term	Meaning
AZ	Availability Zone
VPC	Virtual Private Cloud
TGW	Transit Gateway
DX	Direct Connect
GA	Global Accelerator (as in "NLB/GA")
NLB	Network Load Balancer
ENI / ENA	Elastic Network Interface / Elastic Network Adapter
EC2 / EKS / ECR	Elastic Compute Cloud / Elastic Kubernetes Service / Elastic Container Registry
S3 / EBS / EFS / RDS	Simple Storage Service / Elastic Block Store / Elastic File System / Relational Database Service
AMI	Amazon Machine Image
ODCR	On-Demand Capacity Reservation
IAM / KMS	Identity and Access Management / Key Management Service
IMDSv2	Instance Metadata Service, version 2
MSK	Managed Streaming for Apache Kafka
AMP / AMG	Amazon Managed Service for Prometheus / Amazon Managed Grafana
FIS	Fault Injection Service
SOCI / OCI	Seekable OCI (lazy image loading) / Open Container Initiative
Kubernetes and platform
Term	Meaning
CNI / CSI	Container Network Interface / Container Storage Interface
CA	Cluster Autoscaler
PDB	PodDisruptionBudget
DRA	Dynamic Resource Allocation
CFS	Completely Fair Scheduler (Linux CPU quota/throttling)
NUMA	Non-Uniform Memory Access
SR-IOV / VF	Single Root I/O Virtualization / Virtual Function
DPDK	Data Plane Development Kit
BGP / VIP / PKI	Border Gateway Protocol / Virtual IP / Public Key Infrastructure
VRF / VLAN	Virtual Routing and Forwarding / Virtual LAN
IGMP / PIM	Internet Group Management Protocol / Protocol Independent Multicast
RDBMS / TSDB	Relational / Time-Series Database
OOB / MGMT	Out-of-band / management network
IPMI	Intelligent Platform Management Interface
KVM	Keyboard-Video-Mouse switch here, not the hypervisor
Facility, operations and compliance
Term	Meaning
DC	Data centre (also the "DC-n" challenge IDs)
UPS / PDU / PSU / ATS	Uninterruptible Power Supply / Power Distribution Unit / Power Supply Unit / Automatic Transfer Switch
CRAC / CRAH / DLC	Computer Room Air Conditioner / Air Handler / Direct Liquid Cooling
RMA	Return Merchandise Authorization
HA / DR	High Availability / Disaster Recovery
SLO / SLA	Service Level Objective / Agreement
RTT	Round-trip time
QC / QA	Quality Control / Quality Assurance
GC	Garbage collection (as in "GC/IO stall")
TCO	Total Cost of Ownership
GDPR / DPIA	General Data Protection Regulation / Data Protection Impact Assessment
TPN	Trusted Partner Network (media-industry security assessment programme)
SOC 2 / ISO 27001	Security and compliance audit standards
Ambiguities worth knowing about
MIG: Multi-Instance GPU in the technical sections, migration in the register IDs (MIG-1…12).
DC: data centre in prose, and the ID prefix for current-architecture challenges.
N: concurrent matches in §0.1, a gate placeholder in §5.
ISO: isolated camera feed in the signal-flow sections, ISO 27001 in §3.12.
GA: Global Accelerator, not general availability.
