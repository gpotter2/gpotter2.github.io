---
layout: post
title:  "How does video work on Hyper-V, and how the hell do I get 'Enhanced Session' to actually work properly on Linux?!"
date:   2026-05-09 00:00:00 +0200
categories: hyper-v rdp xrdp
permalink: /blog/2026-hyperv-video
---

Sometimes, you want to run some Linux code that you don't really trust and need the isolation that a full-fledged Hyper-V VM provides. I've always been annoyed at how worse the experience becomes when you start doing so compared to what WSL has to offer: you now need to follow one of the gazillion guides that explain "how to make Hyper-V Enhanced Session work" because you want that sweet copy & paste feature, but end up copy pasting some random unexplained "disable tls security" stuff... I suspect that most of those guides simply copy/pasted some derivative of [Microsoft's original instructions](https://github.com/microsoft/linux-vm-tools/), back when they provided first-party Ubuntu VMs and [still cared](https://techcommunity.microsoft.com/blog/virtualization/sneak-peek-taking-a-spin-with-enhanced-linux-vms/382415), but don't actually understand a thing about the internals. I think it would be pretty cool to understand how all of that actually works and generally what the hell is going on when one starts `vmconnect.exe`. Or WSL for that matter. *Okay, let's do this one last time, yeah? For real this time.*

# tl;dr - how do I configure Hyper-V and xrdp ?

In 2026 you need the following. You DON'T need to touch anything related to TLS, RDP encryption or whatnot.

- install xrdp v0.14 or later, configure `/etc/xrdp/xrdp.ini` :
```
port=vsock://1:3389
vmconnect=true
```
- (sound) install [`pipewire-module-xrdp`](https://packages.debian.org/sid/pipewire-module-xrdp) (debian), for other plateforms: https://github.com/neutrinolabs/xrdp/wiki/How-to-set-up-audio-redirection
- On HyperV: `Set-VM -EnhancedSessionTransportType HvSocket <VMNAME>`

# How video works on Hyper-V

You opened `vmconnect.exe`, you see picture. What happened behind the scenes? Well... it depends. On a bunch of parameters actually, because Hyper-V has had a pretty long life and so had its video components. The main three things that impact the flow are:

- the VM generation;
- the OS;
- whether "Enhanced Session" or "Basic Session" is used.

The issue with "Basic Sessions" is that they are not super user-friendly. You can't copy/paste text nor files, can't resize the screen, can't use a webcam and so on. That's why Microsoft provided an upgrade to that with the "Enhanced Session" mode, which works completely differently.

|   Server-side      | VM gen 1               | VM gen 2                     |
|--------------------|------------------------|------------------------------|
| "Basic Session"    | [Emulated S3 controller](https://learn.microsoft.com/en-us/windows/win32/hyperv_v2/msvm-s3displaycontroller) | [Synthetic display controller](https://learn.microsoft.com/en-us/windows/win32/hyperv_v2/msvm-syntheticdisplaycontroller) |
| "Enhanced Session" | RDP                    | RDP                          |

On a **Basic Session**:
  - **Generation 1** The Hyper-V host uses an "Emulated S3 controller" (`vms3cap.sys`). It's an entirely emulated video card that is detected as actual hardware by your VM. This required no adaptation from an OS to Hyper-V, but it has the downside of requiring a bunch of emulation which impacts not only performance but also increases "the attack surface", as Microsoft [puts it](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/plan/Should-I-create-a-generation-1-or-2-virtual-machine-in-Hyper-V#whats-the-difference-in-device-support).
  - **Generation 2** VMs use a "Synthetic display controller", which is "software based" meaning that it requires a specific driver inside the VM kernel. Hyper-V uses the "VMBUS" (a communication channel between the Host and the VM, see [this](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/architecture#vmbus-vsp-and-vsc)) to talk to the VM whose kernel is responsible for creating a virtual display adapter.
    - On Windows, this has obviously been integrated day-1 inside the `HyperVideo.sys` driver ("Microsoft VMBus Video Device Miniport Driver") driver.
    - On Linux, this started as an ["integration pack"](https://github.com/microsoft/LIS3.5/tree/master) before getting contributed by Microsoft to the linux kernel back in 2012 as the ["Hyper-V Frame Buffer driver" (or hyperv_fb)](https://github.com/torvalds/linux/blob/68eeb0871e986ae5462439dae881e3a27bcef85f/drivers/video/fbdev/hyperv_fb.c). This driver was [replaced in 2021](https://github.com/torvalds/linux/commit/40227f2efcfb7148fdee8d69ba52301951ef8a32) by a DRM-compatible variant ([hyperv_drm](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/hyperv)), but essentially works in the same way as it did back in 2012. It's built on an [implementation of the VMBUS](https://github.com/torvalds/linux/blob/68eeb0871e986ae5462439dae881e3a27bcef85f/drivers/hv/vmbus_drv.c) bundled in the linux kernel, also contributed by Microsoft, very similarly to Windows.

On an **Enhanced Session**:
  - A RDP server needs to run inside the VM. It sends video but also allows for integration services, copy/pasting, sharing devices, etc.
  - On Windows, this is implemented directly by 'Remote Desktop Services' that listens directly on the VMBUS.
  - On Linux, this has **no implementation in the Linux Kernel**. This is implemented in userland, typically with a server like `xrdp` (typical for Hyper-V) or `weston` (WSL). Because those are in userland, they can't easily use the VMBUS and therefore bind a special kind of socket (`AF_HYPERV` / `AF_VSOCK`) on port 3389, which provides a socket-like mechanism over VMBUS. Note that while `AF_VSOCK` uses ports, `AF_HYPERV` uses GUIDs, so 3389 is [actually converted into](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/make-integration-service#register-a-new-application) `00000d3d-facb-11e6-bd58-64006a7986d3`.

What about the client side? What actually comes out of `vmconnect.exe`? RDP. It's only RDP.

![](/assets/posts/2026-04-06-enhanced-hyperv-xrdp/rdp-always-has-been.png)

The connection flow is as follows:
- The `vmconnect.exe` connects to the Hyper-V Virtual Machine Management service `vmms.exe` (yeah, the same that manages the VMs, handles Hyper-V WMI commands, etc.) on port TCP/2179. The fact that it's a TCP port allows `vmconnect.exe` to connect remotely.
- The `vmconnect.exe` sends a special blob called the PCB ("Pre-Connection Blob") that contains the GUID of the VM to connect to. This blob is specific to Hyper-V RDP, you wouldn't find it in a connection generated from a `mstsc` to a remote Windows host. `vmms.exe` looks at the GUID to find the VM the user wants to connect to, but also handles authentication. When the "Enhanced Session" mode was added, the PCB was extended to have an extra `;EnhancedMode=1` suffix which indicates whether "Enhanced Session" is used or not.
- If `EnhancedMode` is 0, then the RDP terminates in `vmms.exe` and Hyper-V feeds it the frames provided by the display controller of the VM matching the GUID in the PCB. If `EnhancedMode` is 1, it forwards the RDP stream to the VM so that it handles it by itself.
- Modern RDP clients wrap their connection in TLS. What's a bit weird in this setup is that the TLS needs to happen between the `vmconnect.exe` client and `vmms.exe` as it handles the authentication. It wouldn't make sense to have to re-TLS the RDP stream between `vmms.exe` and the RDP server. The server therefore needs to be configured in a special mode, where it *says* that it supports TLS during the RDP negotiation, even though TLS actually never arrives because it's eaten by `vmms.exe`.

<p align="center">
  <img src="/assets/posts/2026-04-06-enhanced-hyperv-xrdp/rdp-hv-vmms.png" alt="Centered image">
  <br/>
  <a href="https://github.com/neutrinolabs/xrdp/pull/3514">PR #3514 from xrdp</a> showing the TLS is only between the client and vmms.exe
</p>

`xrdp` (nor most RDP servers on Linux) did not implement this special edge-case, which meant that it crash when receiving unencrypted trafic after having advertised TLS. This is why so many guides disable TLS in the server, leading the client to fallback to "RDP security", which doesn't have this edge case. This however means that setting the GPO to enforce TLS security (as you should) would break Enhanced Sessions on Linux. I ended up [contributing a PR](https://github.com/neutrinolabs/xrdp/pull/3514) back in 2025 which ended up in xrdp v0.10.4.0 to add support for this mode, meaning you no longer mean to force the legacy "RDP encryption". This took the form of adding the following attribute to the configuration.

```
vmconnect=true
```

# (Bonus) Use alternative RDP clients to connect to Hyper-V VMs

### **vmconnect.exe**

Nothing to be said. Did you know it existed?

### **mstsc.exe** (native Windows)

1. Get the VM id through `Get-VM -Name <Your VM Name> | fl Id`
2. Create a `.rdp` file with at least:
```
full address:s:<Hyper-V machine name. Use FQDN so that kerberos works>
pcb:s:<Hyper-V VM ID>
server port:i:2179
negotiate security layer:i:0
sawvmconnect:i:1
```

A few notes:
- `pcb` is the "PCB", sent before the connection is actually established as explainer before
- `sawvmconnect` is some reverse-engineered magic parameter I found that sets the used SPN to "Microsoft Virtual Console Service/XXXX". It also disables CredSSP, so it won't prompt for a password and use Kerberos (hurrah). This should help a lot in hardened environments.

### **msrdc.exe** (Windows App / WSL / Microsoft Remote Desktop)

1. Same as for `mstsc.exe`;
2. `sawvmconnect:i:1` isn't implemented. In a hardened environment, you can work around that by setting the GPO
  > Computer Configuration > Administrative templates > System > Credentials Delegation >
  > "Restrict delegation of credentials to remote servers" = Restrict Credential Delegation (at least, RestrictedAdmin also works but is more restrictive)

It will have an impact on you other usages of RDP which might be an issue in corporate environments (you will need to enable RestrictedAdmin/Remote Credential Guard on all other machines via GPO, or your client won't be able to connect anymore). This is good in terms of security though.

