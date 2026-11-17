---
title: "Windows 11 Reverse Engineering Confirmed: MissionControlAggregator Hardcodes Coding Agent Telemetry and How to Block It"
description: "Dissecting Windows 11 Build 26100 and MissionControlAggregator.dll: how Microsoft quietly fingerprints Claude Code, Cursor, Codex, and terminal tools via ETW events, plus the full SOP to disable telemetry via registry, group policy, and service shutdowns."
pubDate: 2026-11-17
author: "Xiao Ke"
category: "Architecture"
tags: ["Windows11", "CodingAgent", "ReverseEngineering", "Privacy", "Telemetry", "Security"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/windows-11-mission-control-coding-agent-telemetry-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Windows 11 Reverse Engineering Confirmed: MissionControlAggregator Hardcodes Coding Agent Telemetry and How to Block It

A few nights ago, I ran an automated refactoring script locally.

Midway through the run, the chassis fan spun up to maximum speed.

I pulled up Task Manager to check resource utilization.

CPU usage was sitting low at around 8%.

Yet disk write throughput had spiked suddenly to almost 20MB/s.

I avoided jumping to conclusions.

I fired up Process Monitor to inspect the event stream.

I set the filter to match process names containing svchost.exe and DiagTrack.exe.

The trace logs showed Windows diagnostic telemetry hammering several temporary tracking files.

I opened an administrative command prompt and ran:

```cmd
sc stop DiagTrack
```

The moment the command reported success, disk writes dropped back to 0MB/s.

Two minutes later, the fans spun down from 3800 RPM to a silent 1600 RPM.

When I restarted the service, the heavy writes resumed immediately.

The cause and effect were clear.

What was the diagnostic service logging so aggressively late at night?

Yesterday on Hacker News, independent security researcher Xu Sheng published a reverse engineering audit.

The discussion thread quickly drew over five hundred upvotes.

The original analysis writeup is here: https://xusheng.dev/posts/windows-coding-agent-telemetry/

You can inspect the full breakdown there.

The community discussion thread is here: https://news.ycombinator.com/item?id=50524736

Both sources match down to the exact strings.

The author caught Microsoft scanning for developer tooling and coding agents inside recent Windows 11 preview builds.

The analyzed samples came from Windows 11 Build 26100 and Build 26200.

I did not take text writeups at face value.

I extracted the binary from my local system directory and verified it in an isolated VM.

Does this hardcoded roster really exist in the binary?

I ran the strings utility from Sysinternals against the read-only section:

```cmd
strings -el %SystemRoot%\System32\MissionControlAggregator.dll | findstr /i "claude codex cursor copilot"
```

The terminal immediately dumped plain text identifiers.

The strings matched the report completely.

---

## 1. Binary Forensics: The Hardcoded Roster Inside a System DLL

The entry point sits deep inside the Windows 11 system directory.

The binary is located at C:\Windows\System32\MissionControlAggregator.dll.

Its official metadata claims it is merely a task and system metrics aggregator.

We loaded this DLL into Binary Ninja and navigated straight to the read-only data section.

The detection strings sit in uncompressed, wide-character plaintext.

No unpacking required.

No runtime obfuscation at all.

<img src="https://img.freeaidog.com/images/blog/windows-11-coding-agent-telemetry-binary-ninja.webp" alt="Binary Ninja reverse engineering showing hardcoded coding agent processes inside MissionControlAggregator.dll" />

Opening the disassembly view reveals two distinct matcher slots.

The first slot targets command-line AI coding agents.

The binary explicitly checks for four executable names:
- copilot.exe
- claude.exe
- codex.exe
- agy.exe

What happens if the developer works inside WSL?

The matching logic strips the exe extension automatically.

It looks specifically for native Linux binaries launched inside WSL.

The second slot covers desktop coding assistants and IDEs across five targets:
- github.exe
- claude.exe
- codex.exe
- cursor.exe
- antigravity.exe

Beyond these recent AI coding tools, the matcher includes a broad list of traditional toolchains.

It checks for gcc.exe.

It matches clang.exe.

gdb.exe, javac.exe, and dotnet.exe are all present.

python.exe and node.exe are monitored as well.

Visual Studio devenv.exe is listed.

VS Code code.exe is also tracked.

Microsoft even verifies digital signatures for selected common runtimes.

For instance, when node.exe triggers, it verifies the official Node.js signature.

When python.exe triggers, it checks the Python Software Foundation signature.

Open source compilers like GCC and Clang are recorded whenever the process name matches.

The internal feature flag governing this scanner is named CLICodingAgentUsage.

Its internal feature identifier is 62580080.

Why does a low-level operating system telemetry component care which AI coding companion you run?

Is this really necessary data for diagnosing system kernel crashes?

Let us inspect the underlying event subscribers.

---

## 2. Telemetry Pipeline: From Kernel Events to ETW Dispatch

I observed no persistent high-CPU polling loop.

In Process Monitor and kernel call stacks, this mechanism is entirely event-driven.

During Aggregator::OnLoad initialization, the module subscribes to existing Windows kernel event channels.

The first channel monitors native Windows processes.

It subscribes to KernelProcess.AppStateChangeSummary.

It also subscribes to the UI event Win32kTraceLogging.AppInteractivitySummary.

Whenever a process launches, suspends, gains focus, or exits, the kernel emits an app state summary.

The aggregator sits on this bus, receives the notification, and compares the image name against the target list.

The second channel targets Linux developers.

It subscribes to the WSL management channel Microsoft.Windows.Lxss.Manager.Exec and ExecCritical.

This is a kernel probe for the subsystem.

The moment you run claude or codex inside WSL, the driver reports the imageName parameter to the host.

Detection occurs instantaneously.

When an executable matches the criteria, the module invokes a callback shaped like std::_Func_impl_no_alloc.

A match does not immediately initiate network egress.

Instead, the module records the result into an in-memory aggregation bitmap.

The event architecture looks like this:

```text
Aggregator::OnLoad Subscriptions
├─ WSL Subsystem Environment
│  └─ Lxss.Manager.Exec Event → CLI Agent Matcher [copilot, claude, codex, agy]
├─ Native Windows Environment
│  └─ KernelProcess.AppStateChangeSummary → CLI Agent Matcher
└─ Desktop UI Interactivity
   └─ Win32k.AppInteractivitySummary → Desktop Agent Matcher [cursor, claude, codex]

Aggregator::OnRelativeTimerExpire Periodic Timer
└─ MissionControlAggregatorLogging::ReportCohorts<12,4,5>
   └─ Compressed into Segments.SubCohorts bitmap
      └─ Emitted as ETW event for DiagTrack logging
```

Once gathered, an internal relative timer triggers periodically.

The function ReportCohorts<12,4,5> compresses the period observations into a compact bitmap.

It emits an ETW TraceLogging event named Segments.SubCohorts.

What do the numbers 12, 4, and 5 signify?

They correspond to twelve standard developer tools, four CLI agent flags, and five desktop agent flags.

The system avoids transmitting full file paths.

It simply flips the corresponding bit to 1.

For example, if Codex executed during the preceding window, its assigned bit is set.

The binary payload writes to disk and is handed to DiagTrack for batched cloud upload.

My reverse engineering followed the call chain up to ReportCohorts.

Deeper cloud-side aggregation and profiling belong to closed Microsoft backend infrastructure, so we avoid speculating.

---

## 3. Opt-in by Default: How Your Daily Usage Leaks

Review the initial Windows setup wizard.

During fresh installs or major feature updates, what is checked by default across most locales?

The default setting is Send optional diagnostic data.

Microsoft service agreements define this clearly: optional diagnostic data includes information about websites browsed, how apps and features are used, and enhanced error reporting.

The CLICodingAgentUsage feature we unpacked hooks directly into this optional diagnostic pipeline.

Unless you manually selected Send only required diagnostic data during setup, the OS arrived pre-authorized to capture these profiles.

For an individual hobbyist, this might look like a harmless aggregate counter.

What about enterprise source code and engineering stack choices?

What about peak development hours?

What about organizational adoption rates of competing AI coding agents?

In high-compliance enterprise environments, this silent telemetry represents an unmonitored external observability leak.

Teams spend massive effort locking down code egress, unaware that the OS layer already fingerprints their AI workflow.

---

## 4. Local Defense: A Three-Tier SOP to Lock Down Telemetry

With the pipeline understood, you can cut the feed in three straightforward steps.

### Step 1: Disable Optional Diagnostics in Settings

Open Windows Settings.

Navigate to Privacy & security, then select Diagnostics & feedback.

Review the Diagnostic data section.

Switch the toggle from Send optional diagnostic data to Send required diagnostic data only.

Turn off Tailored experiences below it.

Click Delete diagnostic data to wipe Microsoft local usage history from the drive.

Developers have noticed a catch: after monthly cumulative updates, can individual UI settings silently reset?

Yes, that risk is real.

Relying solely on the settings app is insufficient, so we apply system-level blocks.

### Step 2: Enforce Hard Blocks via Group Policy and Registry

On Windows Pro, Enterprise, or Education, press Win + R, type gpedit.msc, and open the Local Group Policy Editor.

Navigate to Computer Configuration > Administrative Templates > Windows Components > Data Collection and Preview Builds.

Double-click Allow Diagnostic Data.

Set the policy to Disabled, or set it to Enabled with Diagnostic data off selected from the dropdown.

Save and exit.

What if you run Windows Home edition without gpedit.msc, or want to deploy this across machines via script?

Open an administrative terminal and run these two registry commands:

```cmd
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v MaxTelemetryAllowed /t REG_DWORD /d 0 /f
```

These values enforce the highest priority constraint on Windows diagnostics.

Setting the value to 0 forces telemetry collection into a hard disabled state.

Even if the DLL generates an internal bitmap, the kernel pipeline drops the payload.

Keep the enterprise boundary in mind: if the machine is enrolled in Active Directory or managed via Microsoft Intune, will domain policy override local registry keys?

Yes, it will, meaning organizational IT policies must push this block via Group Policy Objects or Intune configuration profiles.

### Step 3: Shut Down Diagnostic Services and Sinkhole Telemetry Domains

With the pipeline severed, the background dispatch services simply waste memory.

Stop and disable them from an administrative shell:

```cmd
sc stop DiagTrack
sc config DiagTrack start= disabled

sc stop dmwappushservice
sc config dmwappushservice start= disabled
```

If your environment enforces strict network containment, map known telemetry hosts to loopback in your hosts file:

```text
127.0.0.1 v10.events.data.microsoft.com
127.0.0.1 v20.events.data.microsoft.com
127.0.0.1 watson.telemetry.microsoft.com
127.0.0.1 telemetry.microsoft.com
```

Consider another critical edge case: if Windows or applications use DNS over HTTPS [DoH], will static entries in hosts be bypassed?

Yes, DoH bypasses local hosts resolution.

In strict enterprise environments, configure perimeter firewalls to sinkhole these domains and drop outbound connections to related telemetry IP ranges.

---

## 5. Verification and Ongoing Maintenance

Restart your system after applying these modifications.

Once back at the desktop, open an administrative command prompt and run the three verification checks:

```cmd
logman query providers | findstr /i "MissionControl"
sc query DiagTrack
sc query dmwappushservice
```

Inspect the returned status output:
MissionControl tracking sessions report inactive, while DiagTrack and dmwappushservice both report STOPPED with a start type of DISABLED.

Did both services stay completely stopped on your test setup as well?

Monitoring Process Monitor for ten minutes confirms that the aggressive writes to temporary tracking files do not return.

Chassis fan speeds remain quiet at idle.

Will future Windows cumulative updates quietly re-enable these registry settings?

We recommend running this three-command check after each monthly Patch Tuesday to confirm your boundaries remain intact.

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/windows-11-mission-control-coding-agent-telemetry-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

