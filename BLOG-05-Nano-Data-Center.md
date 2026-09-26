# Sovereign Inference at the Clinic Edge: A Realistic Nano Data Centre

*Building Clinical Software That Deserves Trust, part 5*

**For:** CIOs, infrastructure leads and founders considering running clinical AI models on
hardware they own.

---

There are good reasons to run clinical models on your own hardware instead of a cloud API. There
are also good reasons not to, and they are usually left out of the pitch. This post sets out both
sides, then describes the small "nano data centre" we have designed for ClinuxFlow's enterprise
tier: what it's for, what it runs, how it connects, and how it fails.

It replaces an earlier draft that promised a self-healing cluster "structurally impossible to take
offline", with automatic database failover and a voice pipeline. None of that was built, and the
honest version is more useful.

## Why run models on owned hardware

**Data residency and exposure.** A consultation transcript sent to a third-party API is a
transfer of sensitive personal data, with contracts, retention questions and a lawful-basis
obligation under India's DPDP Act. Inference on hardware the clinic group controls removes the
third party.

**The models you need may not be hosted.** Google's MedGemma (medical text and image models) and
MedSAM (medical image segmentation) are open-weight models built for medical use. Neither is in
the serverless catalog our cloud API runs on, and both need GPU-class memory.

**Predictable cost.** Per-token pricing makes heavy use expensive and hard to budget. Owned
hardware is a fixed cost, which suits a clinic group better than a variable bill.

**Latency and connectivity.** Inference on the local network works when the internet doesn't.

## Why not

- **You are now an operator.** Patching, monitoring, backups, spares and on-call are your job.
- **Power and environment.** Voltage fluctuation, outages, heat and dust are ordinary in many
  Indian clinics. Hardware needs a UPS sized for a clean shutdown, surge protection and airflow.
- **Physical security.** A box in a back room holding clinical data can be carried out of the
  door. Disks must be encrypted.
- **Capacity doesn't stretch.** A GPU serves a fixed number of requests at a time. Ten clinics
  sharing one machine will queue.
- **Model governance is on you.** Which version is running, whether its licence permits your use,
  whether it was evaluated before an upgrade, and how to roll back.

If these costs aren't worth it for a given customer, the cloud path stays. That's why this is an
opt-in enterprise tier, not the default.

## The design

### Hardware

| Node | Role | Notes |
|---|---|---|
| Mac mini, 24 GB unified memory | Clinical text models (MedGemma) | Unified memory lets the GPU use most of RAM. The 4B model fits comfortably at 8-bit; the 27B model at 4-bit is tight once you add context cache, so measure before promising |
| Five single-board computers under Proxmox | CPU-bound clinical NLP (entity extraction, section detection, negation) | Cheap, low power, easy to replace; not for large models |
| Windows PC with an NVIDIA GPU | Imaging models (MedSAM) | Isolated so heavy imaging jobs can't starve text inference |

Splitting by workload means a burst of image segmentation cannot slow down note drafting, and a
failed node takes out one capability, not all of them.

### Connectivity: no open ports

The cluster sits on its own switch with no public IP. It reaches the outside world through a
**Cloudflare Tunnel**: a small agent inside the network makes an outbound connection to
Cloudflare, and requests come back down that connection. No inbound firewall ports are opened.
Access through the tunnel is restricted to our API using service credentials.

### A dedicated gateway

Our cloud API doesn't talk to the models directly. A separate gateway service is the only thing
holding credentials for the nano data centre. It translates our request format into each model
server's own, applies timeouts and a queue, and logs access. It follows the same pattern we use
for the national health registries (part 8). This gateway is designed but not yet written.

### Scheduling

Requests fall into two classes: **interactive** (a clinician waiting for a draft) and **batch**
(re-processing documents overnight). Interactive requests jump the queue; batch fills idle time.
When several clinics share one cluster, each gets a fair share, so one busy clinic can't lock out
the others. This must be designed before a second clinic is added, not after the first complaint.

### Failure is normal, so plan for it

The earlier draft of this post claimed the system could not be taken offline. Every system can.
What matters is what happens when it is:

1. **Clinical work never waits on AI.** If the nano data centre is unreachable, every screen still
   works; AI suggestions are simply absent, and the UI says so. This is the most important design
   rule in this post.
2. **Processes restart themselves.** Model servers run under the operating system's service
   supervisor (systemd on Linux, launchd on macOS) with health checks, so a crash from running out
   of memory on a long input becomes a restart, not an outage.
3. **Requests time out and fail loudly.** A request that doesn't finish in its budget returns an
   error the app can show, rather than hanging.
4. **Nothing clinical is stored only on the cluster.** The models are stateless; records live in
   the app's own storage tiers (part 3), so losing a node loses no patient data.

### Data handling

- Traffic carries clinical text and images, so it is encrypted in transit end to end.
- Prompts and outputs are not logged by default. Access is logged: who, when, which model, how long.
- Retention on the cluster is measured in minutes, not months.
- Disks are encrypted at rest, and the machines sit in a locked space.

### Model governance

- Models are pinned by exact version and file hash; nothing auto-updates.
- Each model's licence is reviewed for clinical use (MedGemma is distributed under Google's Health
  AI Developer Foundations terms, which have conditions of their own).
- A model upgrade runs the same evaluation set as a code change (part 2), and a rollback is one
  configuration change.

### Commercial shape

Shared (several clinics on one central cluster) or dedicated (a clinic's own cluster) is the same
software with a different endpoint per clinic. Which a customer gets is a pricing decision. Shared
mode is only offered once the scheduling above exists.

## Questions to ask any vendor selling on-premises clinical AI

1. What happens to clinical work when the AI box is off?
2. Which exact model versions run, and how are upgrades evaluated and rolled back?
3. What is logged, and for how long?
4. Who patches it, and how quickly?
5. What's the UPS runtime, and does the hardware shut down cleanly?
6. What throughput do you measure for our workload, not a benchmark?

## Where ClinuxFlow is today

The hardware exists. The design above is written down. Nothing is deployed: no tunnel, no gateway,
no scheduling, and no enterprise-tier switch in the product. This is the last item on our AI
roadmap, deliberately, because the rules in parts 2 and 4 have to be met first. The next post looks
at the conversational workspace these models would eventually sit behind.
