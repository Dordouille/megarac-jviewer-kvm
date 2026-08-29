# megarac-jviewer-kvm

> Get the **Java (JViewer) remote console** of a legacy **AMI MegaRAC** BMC working again — in your **browser, over noVNC, with no Java installed locally**. Verified on an **ASUS ASMB8-iKVM** (ASPEED **AST2400**, firmware 1.14).

![noVNC](https://img.shields.io/badge/console-noVNC-2E7D32)
![BMC](https://img.shields.io/badge/BMC-AMI%20MegaRAC-informational)
![ASPEED](https://img.shields.io/badge/ASPEED-AST2400-informational)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Is this your problem?

Your server's IPMI web UI offers a **"Remote Control → Console Redirection"** button. It downloads a `jviewer.jnlp` file. Then one of these happens:

| Symptom | Cause | Fixed by |
|---|---|---|
| Nothing launches; your OS has no idea what a `.jnlp` is | **Java Web Start was removed** from modern JREs (dropped in Java 11) | running the viewer in a container (below) |
| `Fatal: Read Error: Could not read or parse the JNLP file` (IcedTea-Web) | the BMC returned an **error page** instead of launch arguments | [quirk 1](#quirk-1--the-launch-file-needs-two-query-parameters) |
| The file body reads `Unable to find JNLP String` | same as above | [quirk 1](#quirk-1--the-launch-file-needs-two-query-parameters) |
| The viewer opens but sits at **`Connection in progress`** forever | the KVM stream is **not on port 7578** — the BMC is in single-port mode | [quirk 2](#quirk-2--the-kvm-stream-runs-on-port-80-not-7578) |
| `java.lang.NullPointerException` from netx before anything is drawn | the launch file has **no `main-class`** | [quirk 3](#quirk-3--the-launch-file-has-no-main-class) |
| The BMC's HTTPS side refuses to negotiate (`TLS 1.0`, `RC4-MD5`) | no current JRE will speak that | [quirk 2](#quirk-2--the-kvm-stream-runs-on-port-80-not-7578) — talk HTTP instead |
| The console connects but the screen stays **black** | the transport is fine; the BMC has no video to capture | [Black console](#a-black-console-is-a-different-problem) |

If you recognise two or more of those, you are in the right place.

## What this is

[`nojava-ipmi-kvm`](https://github.com/sciapp/nojava-ipmi-kvm) already does the hard structural work: it runs an old JRE + IcedTea-Web inside a container, logs into your BMC, launches the Java viewer there, and serves the result to your browser over noVNC. Nothing to install on your desktop.

On this firmware generation it still cannot reach the host's video, because of **three firmware quirks**. This repo supplies:

* **`megarac-patch/`** — a small Docker layer over the upstream image that patches its `get_java_viewer` to work around all three (and fails loudly instead of hanging when the BMC misbehaves).
* **`nojava-ipmi-kvmrc.example.yaml`** — a working configuration for this BMC family.
* **This README** — the *why*. Each quirk fails in a way that points somewhere else entirely, so the diagnosis is worth more than the patch.

**Scope, honestly stated:** this is a *workaround wrapper*, not a clean-room KVM client. It still runs AMI's Java viewer — just somewhere you don't have to care about it. If you want a native reimplementation, see [Related work](#related-work).

---

## Quick start

### 1. Install the launcher (pinned — both pins matter)

```sh
# Python 3.13, NOT 3.14 — 3.14 breaks the tool's asyncio.get_event_loop() usage.
# Pin 0.9.2 — 0.9.3 has no matching image on Docker Hub.
pipx install --python python3.13 nojava-ipmi-kvm==0.9.2
```

### 2. Build the patched image

Tag it as the image `nojava-ipmi-kvm` expects, so it is used locally without re-pulling over it:

```sh
git clone https://github.com/Dordouille/megarac-jviewer-kvm.git
cd megarac-jviewer-kvm
docker build --platform linux/amd64 \
  -t sciapp/nojava-ipmi-kvm:v0.9.2-openjdk-7 megarac-patch
```

`--platform linux/amd64` is required on Apple Silicon and other ARM hosts — the base image is amd64-only. The patch is **idempotent**: rebuilding over an already-patched image replaces its own block rather than stacking onto it.

### 3. Configure

```sh
cp nojava-ipmi-kvmrc.example.yaml ~/.nojava-ipmi-kvmrc.yaml
```

Then edit it — three fields, and they must agree:

| Field | Set it to |
|---|---|
| `full_hostname` | your **BMC's** IP or hostname (not the host OS, not a VM) |
| `download_endpoint` | the same IP inside `EXTRNIP=` — **keep it in sync** |
| `login_user` | your BMC account |

The password is prompted at runtime and never written to disk.

### 4. Run

```sh
nojava-ipmi-kvm --debug myserver
```

Open the `http://localhost:<port>/vnc.html?…` URL it prints. **Keep the terminal open** — it holds the container, and closing stdin is read as "shut down".

### 5. Verify the video transport actually came up

Rather than trusting the window:

```sh
docker exec <container> netstat -tn | grep :80    # expect ESTABLISHED to the BMC
```

An `ESTABLISHED` connection to the BMC on **port 80** means the stream is live. Nothing there means you are still stuck on [quirk 2](#quirk-2--the-kvm-stream-runs-on-port-80-not-7578).

---

## The three quirks

### Quirk 1 — the launch file needs two query parameters

`Java/jviewer.jnlp` on its own returns an error page where the launch arguments should be. Both parameters are mandatory:

```
Java/jviewer.jnlp?EXTRNIP=<bmc-ip>&JNLPSTR=JViewer
```

* **`JNLPSTR=JViewer`** — without it the body reads *"Unable to find JNLP String"*.
* **`EXTRNIP`** — copied **verbatim** into JViewer's `-hostname` argument. It must be the **BMC's own address**. Point it at the client by mistake and the BMC cheerfully builds a launch file telling the viewer to connect to the client.

The response also declares `Content-Length: 4134` while sending ~3154 bytes. The document is nevertheless **complete** (it ends in `</application-desc></jnlp>`); only the header is wrong. An HTTP client that treats a short read as fatal will discard a perfectly good file — read the partial body instead (`http.client.IncompleteRead.partial`).

### Quirk 2 — the KVM stream runs on port 80, not 7578

This BMC runs in **single-port mode** (`-singleportenabled 1`). Video redirection and virtual media are multiplexed onto the **web server port**. Ports 7578 / 7582 / 5120 / 5123 answer with TCP RST and are never used — even though *Configuration → Services* still lists them:

```
2  kvm  Active  both  7578  7582  1800  4  View
```

That display is the *stored configuration*; single-port mode overrides it. A viewer aimed at 7578 sits at *"Connection in progress"* forever, which reads like an authentication or privilege problem and is neither.

Which port the BMC hands out depends on **the scheme of the session that asked**:

| Login over | `-kvmsecure` | `-kvmport` |
|---|---|---|
| HTTPS | `1` | `443` |
| HTTP | `0` | `80` |

The BMC's HTTPS side only negotiates **TLS 1.0 + RC4-MD5**, which no current JRE will do — so asking over HTTPS hands the viewer a transport it cannot open. The BMC serves its **entire web UI in the clear on port 80** with no redirect, so asking over HTTP yields a plaintext video stream and the TLS problem disappears. `megarac-patch` therefore forces the container to speak HTTP.

> **Security note.** This means the console stream — keystrokes included — crosses the network **unencrypted**. That is a property of the firmware, not a choice this repo makes for you: its HTTPS is TLS 1.0/RC4, which is not meaningfully better. Treat the BMC as what it is — a device that belongs on an **isolated management VLAN**, reached over a VPN, never exposed to a routable network.

### Quirk 3 — the launch file has no `main-class`

The firmware emits a bare `<application-desc>` and relies on the jar manifest (which does carry `Main-Class: com.ami.kvm.jviewer.JViewer`). This netx build does not cope, and dies with a `NullPointerException` before JViewer's `main()` runs. The patch injects the attribute.

It also repairs the `codebase="http://(null):80/Java"` the firmware emits, without which the jars cannot be resolved.

---

## Operational notes

### The MegaRAC web server is fragile — read this before you retry

It allows roughly **4–5 concurrent web sessions** and **leaks one on each failed console attempt**. Once exhausted it still *accepts* TCP in ~0.00 s but never answers HTTP, while IPMI keeps replying normally — a combination that looks exactly like a network fault and is not.

Recover with a **BMC-only cold reset**. This does **not** touch the running host; it takes 1–2 minutes:

```sh
ipmitool -I lanplus -H <bmc-ip> -U <user> -f <passwordfile> mc reset cold
```

So: keep failed attempts few, and log out (`rpc/logout.asp`) when scripting against the BMC.

### A black console is a different problem

If the viewer connects — traffic settles to a trickle of keep-alives — but the screen stays black, the **transport is fine**. The BMC simply has no video to capture. The AST2400 only digitises the onboard VGA it is wired to, so if something else is driving the display, nothing reaches it.

**Which something else depends on when the screen goes black**, and the two cases have different causes and different fixes.

First, confirm you actually have two VGA-class devices, and note their **PCI bus numbers**:

```
$ lspci | grep VGA                          # on Linux
01:00.0 VGA compatible controller: NVIDIA Corporation TU102 [GeForce RTX 2080 Ti]
08:00.0 VGA compatible controller: ASPEED Technology, Inc. ASPEED Graphics Family

$ esxcli hardware pci list                  # on ESXi — look for Device Class Name
   Address: 0000:01:00.0   Device Class Name: VGA compatible controller
   Address: 0000:08:00.0   Device Class Name: VGA compatible controller
```

#### Black the whole way through, POST included

The **firmware** elected the discrete card as primary display — commonly the one at the lower PCI bus number. The BMC never sees a thing, from power-on onwards.

The fix is in the **BIOS**, not in this repo: set *Primary Display* (also called *Onboard VGA*, *VGA Priority*, or *Primary Video Adapter*) to the **onboard / IGD** device instead of *Auto* or *PCIE*. It needs a reboot.

#### POST and BIOS visible, black once the OS boots

The firmware is fine — it is rendering to the BMC, which is why you can see POST, the BIOS setup and an OS installer. The handover happens later: with two VGA devices present, **the OS or hypervisor picks which one carries its console**, and it picked the other one.

No BIOS change will help here, because the BIOS is already doing the right thing. On a general-purpose Linux the console device can be steered with kernel command-line parameters; on ESXi there is no supported knob for it — the practical answers are to redirect the console to **serial** ([SOL](#sol-serial-over-lan--the-text-console), below) or to manage the host over the network and use the KVM for firmware screens only.

**Which is not the limitation it sounds like.** Firmware screens are exactly what a remote console is *for*: BIOS setup, boot order, a bootable ISO over virtual media, a host that will not boot. All of that renders to the BMC in this case, and all of it is unreachable by SSH.

### The BMC also has SSH

Port 22, OpenSSH 6.0p1, RSA/DSS host keys only — a restricted SMASH-CLP shell:

```sh
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa \
    -o KexAlgorithms=+diffie-hellman-group1-sha1 -o Ciphers=+aes128-cbc \
    <user>@<bmc-ip>
```

Open TCP ports observed on this BMC: 22, 80, 427, 443, 555, 623, 5988, 5989.

---

## SOL (Serial-over-LAN) — the text console

If you only need BIOS and OS text — and serial redirection is enabled in the BIOS — you need none of the above:

```sh
ipmitool -I lanplus -H <bmc-ip> -U admin -E sol activate     # ~. to exit
ipmitool -I lanplus -H <bmc-ip> -U admin -E sol deactivate   # if it gets stuck
```

`-E` reads the password from the `IPMI_PASSWORD` environment variable, keeping it off the process list.

---

## Why not a firmware update, Redfish, or a native client?

* **Firmware.** The AST2400 / ASMB8 generation never received an HTML5 iKVM — that arrived with the AST2500 / ASMB9 boards. The last firmware for this one (1.14.51, 2017) adds nothing here.
* **Redfish.** Absent or too limited on this generation to drive a console.
* **Native client.** It exists, but not for this firmware — see below.

## Related work

* [`sciapp/nojava-ipmi-kvm`](https://github.com/sciapp/nojava-ipmi-kvm) — the upstream launcher this builds on. Use it directly if your BMC is *not* a single-port MegaRAC.
* [`BadCoder1337/rd450x-console`](https://github.com/BadCoder1337/rd450x-console) — a clean-room reimplementation: a single Go binary that speaks IVTP, decodes the ASPEED VQ+JPEG video codec and serves noVNC itself, **no Java at all**. It targets a newer MegaRAC (firmware 2.36) over TLS on 7582, so it would need adapting to single-port-over-80 before it could talk to this BMC — but its `docs/kvm-protocol.md` is the best available description of the wire format.
* `mcxBMCView` — **not** applicable: it drives the HTML5 SPA that this firmware generation does not have.
* [`ipmi-mcp`](https://github.com/Dordouille/ipmi-mcp) — sibling project: an MCP server that drives the same BMC over IPMI (power, sensors, event log) from an AI assistant, behind a fail-safe approval gate.

## Hardware this was verified on

| | |
|---|---|
| Board | ASUS, with the ASMB8-iKVM module |
| BMC chip | ASPEED AST2400 |
| Firmware stack | AMI MegaRAC 1.14 |
| Mode | single-port (`-singleportenabled 1`) |

Other AMI MegaRAC BMCs of the same era (Supermicro, Tyan, ASRock Rack…) are likely to behave the same way, but are untested. Reports welcome — open an issue with your board, BMC chip and firmware version.

## License

MIT — see [LICENSE](LICENSE).
