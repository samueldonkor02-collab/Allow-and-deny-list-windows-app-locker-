# Allow-and-deny-list-windows-app-locker-
Allow List / Deny List Lab (Windows AppLocker) — Built and tested Windows AppLocker rules to block specific apps (Registry Editor, Firefox) using path and file hash conditions, enabled the required Application Identity service, and verified enforcement via PowerShell and the Start menu.

# Allow List / Deny List Lab (Windows AppLocker)

## Overview
This lab was my hands-on introduction to application whitelisting — building an **allow list** and **deny list** using Windows AppLocker on a Windows 10 machine. Instead of just relying on antivirus to catch bad software after the fact, this approach flips the model: only what's explicitly trusted gets to run, and everything else — even legitimate software like a browser — can be locked down if it's not supposed to be there.

I set out to block two specific programs (Registry Editor and Firefox) using two different rule types, then actually proved the block worked instead of just assuming it did.

## Objective
Get comfortable configuring, enforcing, and validating application control policies using tools built right into Windows — a control that shows up constantly in real environments to reduce what an attacker (or a careless user) can execute on an endpoint.

## Environment
- Windows 10 (ASUS Vivobook test machine)
- Local Security Policy (`secpol.msc`)
- Windows PowerShell (Administrator)
- Application Identity service (`AppIDSvc`)

## Tools I Used

| Tool | What It Does | Why I Used It |
|------|--------------|----------------|
| **Local Security Policy (secpol.msc)** | Manages local security settings on a Windows machine | Where AppLocker rules actually get built |
| **Windows AppLocker** | Application control / allow-deny listing engine | Core of the whole lab |
| **Windows PowerShell (Admin)** | Command-line management | Starting services, forcing policy updates, testing enforcement |
| **Application Identity Service (AppIDSvc)** | Background service AppLocker depends on | Found out the hard way that rules do nothing without it running |
| **gpupdate /force** | Forces Group Policy to refresh immediately | Didn't want to wait for the normal refresh cycle |

## What I Did

1. Opened Local Security Policy → `Application Control Policies` → `AppLocker` → `Executable Rules` and looked at the default allow list (Everyone can run stuff in Program Files, Everyone can run stuff in the Windows folder, Administrators can run anything).
2. Added a **deny path rule** for `%WINDIR%\regedit.exe` so Everyone would be blocked from opening the Registry Editor.
3. Added a **deny file hash rule** for `firefox.exe` — instead of blocking by location, I generated a hash of the actual binary (661 KB), which is a tighter control since it still blocks the file even if someone renames it or moves it somewhere else.
4. Ran into my first real gotcha: nothing was actually being enforced yet. Turns out AppLocker depends on a background service, the **Application Identity service**, which wasn't running. Started it with `net start appidsvc`.
5. Forced the new policy to apply immediately with `gpupdate /force` instead of waiting around for the next refresh cycle.
6. Tested it for real:
   - Tried to launch `regedit` from PowerShell → got kicked back with "This program is blocked by group policy."
   - Searched for Registry Editor and Firefox from the Start menu and tried to open them → got the Windows dialog "This app has been blocked by your system administrator."

Seeing that block message pop up after all the setup was honestly the most satisfying part — it meant the policy wasn't just sitting there configured, it was actually doing its job.

## What's in This Repo

```
allow-deny-list-lab/
├── README.md                          # This file
├── lab-report.md                      # Detailed breakdown of findings
├── findings/
│   ├── executable-rules-list.txt      # Final AppLocker rule set
│   └── enforcement-test-results.txt   # Block message evidence
├── screenshots/
│   ├── 01-default-rules.png           # Default AppLocker rules
│   ├── 02-path-rule-creation.png      # Creating the regedit deny rule
│   ├── 03-file-hash-rule.png          # Creating the firefox deny rule
│   ├── 04-appidsvc-start.png          # Starting the Application Identity service
│   ├── 05-gpupdate.png                # Forcing the policy update
│   ├── 06-regedit-blocked-cli.png     # Blocked from PowerShell
│   └── 07-firefox-blocked-gui.png     # Blocked from the Start menu
└── scripts/
    └── verify-appidsvc.ps1            # Quick check that the dependency service is running
```

## Skills I Picked Up
- **Building allow/deny rules in AppLocker**, and understanding the tradeoffs between path-based rules (simple, but easy to get around if a file moves) and file hash rules (more rigid, tied to the exact binary).
- **Group Policy fundamentals** — how settings in Local Security Policy actually become enforced behavior, and how to push them out immediately with `gpupdate /force` instead of waiting.
- **Realizing a security control depends on other moving parts** — AppLocker rules do nothing if the Application Identity service isn't running. That was a good lesson in not assuming something is broken just because it's not working yet; sometimes there's a dependency you haven't started.
- **Reading PowerShell error output** and understanding what it's telling you (`NativeCommandFailed`, `ApplicationFailedException`) instead of just seeing red text and panicking.
- **Actually testing my own work** — configuring a rule isn't the finish line. I tried to break it from two different angles (command line and the Start menu GUI) before I trusted that it worked.
- **Writing this up clearly**, with screenshots, so someone else (or future me) can follow exactly what happened and why.

## How This Applies in the Real World
I didn't want this to just be a checkbox exercise, so here's why I actually think this matters outside of a lab:

A lot of ransomware works by dropping some random .exe onto a machine and running it. If that machine has an allow list in place, that .exe never even gets to start — it doesn't matter if antivirus has never seen it before, because the answer is already "no" by default. That's a pretty big shift in mindset from the reactive "catch it after it runs" approach.

Companies also deal with people installing stuff they shouldn't — random browsers, torrent clients, whatever. Instead of someone from IT walking around checking every machine, a policy like this just quietly stops it from happening in the first place.

Blocking `regedit.exe` specifically is something I've actually seen mentioned as a real-world move for locking down standard user accounts — the registry is the kind of thing you really don't want an attacker (or a curious employee) messing with, and this lab showed me exactly how that gets enforced, not just why it's a good idea.

There's also a compliance side to this that I didn't fully appreciate going in — frameworks like PCI-DSS and NIST actually call out application whitelisting as something organizations are expected to have, so this isn't just a "hacker skill," it's something that shows up in audits too.

And the part that stuck with me most: what I did here on one laptop through Local Security Policy is the exact same mechanism (AppLocker) that gets pushed out to thousands of machines at once through Group Policy in a real company. I was just doing it small, on one machine, so I could actually understand what's happening before trying to do it at scale.

## Where I'm Coming From
I'm making the jump into cybersecurity from a background in **healthcare**. It's a different field on paper, but a lot of the muscle memory carries over — following procedures carefully, protecting sensitive information, staying calm when something isn't working the way it's supposed to. I'm currently studying for **CompTIA Security+** and building labs like this one to get real hands-on reps in, since that's what I'm missing on paper right now compared to my experience.

I'm still early in this, and I know it. I don't have all the answers yet, and I'm not pretending to — but I'm putting in the time to actually build and break things rather than just read about them, and I want that to come through in this portfolio.

## What I Want to Learn Next
- Other AppLocker rule types I haven't touched yet — publisher rules, script rules, packaged app rules
- How AppLocker stacks up against Windows Defender Application Control (WDAC), and when you'd reach for one over the other
- Rolling this out at scale through Group Policy Objects (GPO) in an Active Directory environment instead of one local machine
- Where these block events actually show up in Event Viewer, so I could eventually monitor this stuff like a SOC analyst would
