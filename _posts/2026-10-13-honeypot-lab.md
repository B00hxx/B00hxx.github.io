---
layout: post
title: "Setting a trap for attackers: my first internship, a honeypot lab"
description: "What I learned building a segmented honeypot lab during my first internship at SFM Technologies: GNS3, pfSense, Cowrie and Wazuh, what I'd fix today, and how a setup like this grows into a SOC and a source of data for machine learning."
image: /assets/img/posts/honeypot-lab/cover.png
tags: [security, networking, honeypots]
---

Fourth post, and this one goes back in time. Before LLMs, caching and retrieval, my first internship was in the summer of 2025 at SFM Technologies in Tunis, a telecom consulting and engineering company, under the supervision of Yassine Ben Amor. My degree is telecom-focused, so it makes sense that my first real project was about networks and security.

The project: build a network where attackers are welcome, but only in one very specific place. That place is a honeypot, a fake server whose whole job is to get attacked and write down everything the attacker does. Around it, a firewall keeps the real servers out of reach, and a monitoring platform turns the honeypot's logs into alerts.

Reading my internship report a year later was a funny experience. Some parts made me proud, and some made me wince. So this post covers what I built, what I'd fix today (I'll be honest about those), and something the report only hinted at: how a setup like this can grow into a small security operations center, and why its logs are great material for machine learning.

## What's a honeypot, and why SSH

A honeypot is a decoy. It looks like a real machine, it's reachable from the outside, and it has no legitimate users. That last part is the clever bit: since nobody should ever connect to it, every connection is suspicious by definition. There's no need to separate the bad traffic from the good, because there is no good traffic.

That gives you two things. The decoy can pull attackers away from real systems, and it shows you what they actually do: which passwords they try, which commands they run once they're in, what they download.

I chose SSH as the bait. It's the service everyone uses to administer Linux servers remotely, it usually listens on port 22, and it's one of the most attacked services on the internet, mostly by bots that try username and password pairs all day long.

The honeypot itself is [Cowrie](https://github.com/cowrie/cowrie), an SSH and Telnet honeypot written in Python. In its default mode it's a medium-interaction honeypot: it doesn't give the attacker a real shell. It emulates one, with a fake filesystem and fake command outputs, while logging every login attempt and every keystroke. The attacker thinks they've broken into a Linux server. They're typing into a Python program that writes everything down.

## The lab

Everything ran on my laptop. Each machine is a VirtualBox virtual machine, and GNS3 (a network emulator) wires them together with emulated routers and switches, so the whole topology behaves like a real network.

![Attacker and ISP router outside; pfSense in the middle; a honeypot zone with Cowrie and the sensor; a LAN zone with Windows Server](/assets/img/posts/honeypot-lab/topology.png)

The design is adapted from a research paper by Lee and Nam (ETRI) on a P2P honeypot network, which separates a service network from a honeynet watched by a sensor. My version has three zones:

- Outside: a Kali Linux machine plays the attacker, and a Cisco 3725 router emulated in GNS3 plays the ISP, standing in for the internet.
- The LAN, `10.0.1.0/24`: the "real" company services. Here that's a Windows Server with Active Directory, the directory service that manages users, computers and permissions in a Windows network. It's also one of the first things an attacker goes after, because owning it means owning the whole domain.
- The honeypot zone, `10.10.10.0/24`: the Cowrie honeypot (`10.10.10.100`) and a sensor machine (`10.10.10.50`) that collects and analyses its logs.

A pfSense firewall sits in the middle with one interface per zone: WAN (`10.0.0.135`), LAN (`10.0.1.1`) and OPT1 (`10.10.10.1`), which is pfSense's default name for an extra interface. All the addresses are private, but the WAN side is treated as if it were public, so the firewall rules are written exactly as they would be for a machine exposed to the internet.

## The firewall: one door, and it leads to the trap

The firewall's job is to make sure that the only thing reachable from outside is the honeypot, and that the honeypot can never reach the real services.

![The firewall rules: only port 22 is forwarded in, LAN is hidden from the WAN, the honeypot zone can reach everything except the LAN](/assets/img/posts/honeypot-lab/firewall-rules.png)

On the WAN side, there are three rules: allow ping to the WAN subnet (only for testing), block everything going to the LAN, and pass the forwarded SSH traffic to the honeypot. The forwarding itself is a NAT rule: anything that hits the firewall's WAN address on TCP port 22 is redirected to `10.10.10.100:22`.

On the honeypot side, the OPT1 rule lets the honeypot zone reach any destination except an alias called `Private_IPv4s`, which holds the LAN range. So the honeypot can talk to the internet but never to the real servers. In the lab, that alias only contains `10.0.1.0/24`. In a real deployment it should contain all three private ranges (`10.0.0.0/8`, `172.16.0.0/12` and `192.168.0.0/16`), so the honeypot can't reach any internal network at all.

One more detail from the setup: pfSense can run its own SSH server for administration. That has to be off, or it competes with the port-forward and exposes the firewall itself.

From the attacker's point of view, an Nmap scan of the WAN address shows exactly one open port: 22. Which is the point. The only thing an attacker can find is the trap.

## Setting up Cowrie

Cowrie runs as an unprivileged user called `cowrie`, inside a Python virtual environment, from a clone of its GitHub repository. Configuration lives in `etc/cowrie.cfg`, which starts as a copy of the shipped `cowrie.cfg.dist`.

By default, Cowrie listens on port 2222, because only root can bind ports below 1024 on Linux, and you really don't want a honeypot running as root. There are two usual ways around that: redirect port 22 to 2222 with the firewall, or use `authbind`, a small utility that lets a specific non-root user bind a specific low port. I used authbind: create `/etc/authbind/byport/22`, give it to the `cowrie` user, turn on `AUTHBIND_ENABLED` in Cowrie's start script, and point Cowrie at port 22.

```ini
[ssh]
listen_endpoints = tcp:22:interface=0.0.0.0
```

After that, `bin/cowrie start`, and the log says `CowrieSSHFactory starting on 22` and `Ready to accept SSH connections`.

Cowrie writes two logs: a human-readable `cowrie.log` and `cowrie.json`, where every event is one JSON object on its own line. The JSON one is the important one, because it's what the monitoring side reads.

## Watching the trap: Wazuh

A honeypot that nobody watches is a very elaborate log file. To turn its events into alerts, I used [Wazuh](https://wazuh.com/), an open-source security platform. It works like this: a Wazuh agent runs on each monitored machine and ships its logs to a Wazuh manager, which decodes them, matches them against rules, and raises alerts in a web dashboard. That's the role of a SIEM (security information and event management): collect logs from everywhere, and turn them into something a human can act on.

![An SSH attempt goes through pfSense's port-forward to Cowrie, becomes JSON events, and reaches the Wazuh manager through the agent](/assets/img/posts/honeypot-lab/ssh-path.png)

The manager runs on the sensor machine and listens on port 1514 for agents (1515 is for enrolment, and 55000 is its API). The agent runs on the honeypot. Telling it to read Cowrie's log is a few lines in `/var/ossec/etc/ossec.conf`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/home/cowrie/cowrie/var/log/cowrie/cowrie.json</location>
</localfile>
```

Because the format is JSON, Wazuh decodes every field automatically: `eventid`, `src_ip`, `username`, `password`, `input` and so on. Then the rules decide what deserves an alert. I wrote two:

```xml
<group name="HoneypotLogin,">
  <rule id="100200" level="11">
    <decoded_as>json</decoded_as>
    <field name="eventid">cowrie.login.failed</field>
    <description>$(src_ip) tried to login to honeypot.</description>
  </rule>
</group>

<group name="HoneypotConnect,">
  <rule id="100201" level="7">
    <decoded_as>json</decoded_as>
    <field name="eventid">cowrie.session.connect</field>
    <description>$(src_ip) is trying to connect.</description>
  </rule>
</group>
```

Rule 100200 fires on every failed login, rule 100201 on every new connection, and both show the source IP in the description. After that, every test login showed up in the dashboard's threat-hunting view as a "tried to login to honeypot" event.

## Attacking my own network

With everything in place, I switched sides and attacked the lab from the Kali machine.

First, a brute-force attack with Hydra, a tool that tries username and password combinations against a login service. I used a public password list from Kaggle and usernames that sound privileged (`root`, `admin`), with `-R` to resume the attack when the connection dropped. On one side of the screen, Hydra kept guessing. On the other, Cowrie's log scrolled with every attempt, and the alerts landed in Wazuh.

Then the Nmap scan from earlier: one open port, 22, on the WAN address.

Finally, the fun part: logging into the honeypot and poking around like an attacker would. `ls`, `cd`, `pwd`, `whoami`. Cowrie answered all of them convincingly, and every command showed up in its log next to my IP address. That's the moment the whole thing clicked for me: this is exactly the view a defender wants of someone who got in.

## What I'd fix today

Reading my report a year later, a few things stand out. Some are mistakes, some are simplifications that were fine for a lab and wouldn't be fine anywhere else.

The sensor sits in the same subnet as the honeypot. Both are on the same switch in `10.10.10.0/24`, so traffic between them never goes through pfSense, and no firewall rule can filter it. Cowrie's emulated shell makes a real breakout unlikely, but a honeypot is the one machine you have to assume will be compromised. The monitoring box should be on its own network, with the firewall only allowing the agent's connection to the manager on port 1514.

The honeypot has a real SSH server too. The setup installs and enables OpenSSH on the honeypot VM, then asks Cowrie to listen on the same port 22. Two programs can't bind the same port, so the real `sshd` should be moved to another port that's only reachable from a management network, as Cowrie's own documentation recommends. Otherwise, depending on which one starts first, the attacker might talk to the real SSH server instead of the trap.

The honeypot zone can reach the whole internet. That's useful, since Cowrie fetches the files attackers try to download with `wget` or `curl`, which is some of the most valuable data a honeypot collects. But it also means a compromised honeypot could be used to attack someone else. Outbound traffic should be limited and logged.

The alert rules are noisy and miss the important event. A brute-force attack produces thousands of failed logins, and rule 100200 raises a level-11 alert for each one, so the dashboard drowns. Meanwhile, nothing alerts on `cowrie.login.success`, which in a honeypot is the more interesting event: it means someone guessed a password that works and is about to run commands. A better set of rules alerts on a burst of failures from one source, on any successful login, and on commands, and tags them with MITRE ATT&CK techniques so they slot into a standard framework:

```xml
<group name="cowrie,">
  <!-- 8 failed logins from the same source within 60 seconds -->
  <rule id="100202" level="10" frequency="8" timeframe="60">
    <if_matched_sid>100200</if_matched_sid>
    <same_field>src_ip</same_field>
    <description>Brute force on the honeypot from $(src_ip).</description>
    <mitre><id>T1110</id></mitre>
  </rule>

  <rule id="100203" level="12">
    <decoded_as>json</decoded_as>
    <field name="eventid">cowrie.login.success</field>
    <description>$(src_ip) logged into the honeypot as $(username).</description>
    <mitre><id>T1078</id></mitre>
  </rule>
</group>
```

`same_field` is the right tool here, because `src_ip` is a field Wazuh decoded from Cowrie's JSON rather than one of its built-in fields. The individual failed logins can then drop to a low level, so they're stored and searchable without paging anyone. The exact thresholds would need tuning against real traffic, and Wazuh's Ruleset Test tool is the place to check them.

A honeypot doesn't stop attacks. My report's conclusion says the setup prevents attackers from reaching the main services. That's true, but it's the firewall and the segmentation that do it, not the honeypot. The honeypot's value is visibility.

And two small things: the report says Windows Server 2022 in one place and 2025 in another, and it writes the honeypot interface as "OTP1" instead of pfSense's OPT1. The kind of thing you only notice a year later.

## Cowrie versus T-Pot

Cowrie wasn't the only option. [T-Pot](https://github.com/telekom-security/tpotce) is a whole honeypot platform: more than 20 honeypots in Docker containers (Cowrie among them), with an Elastic-based dashboard, live attack maps, and integrations like Suricata for network intrusion detection. It's much more capable, and it's also much heavier: it asks for at least 8–16 GB of RAM and 128 GB of disk, on a laptop already running five VMs and a network emulator. Cowrie did the one job I needed with almost no resources. For a lab, that was the right trade.

## From a lab to a SOC

What I built is a small piece of something bigger: a security operations center, or SOC. A SOC is the team (and the tooling) that watches an organisation's systems all the time, detects attacks, and responds to them. Its core loop is sense, detect, respond, and every piece of my lab maps onto it.

![Honeypots, firewall logs and real servers feed a SIEM, which drives triage, active response, threat intelligence and new detection rules](/assets/img/posts/honeypot-lab/soc.png)

To sense, you need more sources than one honeypot. pfSense can send its firewall logs to Wazuh over syslog, and the Windows Server should have its own Wazuh agent, since it's the machine that actually matters. More honeypots, or T-Pot, would widen what the trap can catch.

To detect, the SIEM correlates across sources. A source IP that brute-forces the honeypot and then shows up in the firewall logs knocking on other ports tells a much clearer story than either event alone.

To respond, Wazuh has active response: when a rule fires, the manager can run a script, and the built-in `firewall-drop` blocks the source IP. The bigger win is threat intelligence. Every IP that touches the honeypot is hostile by definition, and so is every password it tries and every file it downloads. Those indicators can go into a pfSense alias that blocks them on the real network. That's when the honeypot actually starts protecting something: what it learns from attackers who fell for the trap is used against them everywhere else.

And the last loop is detection engineering: reading what attackers actually did, and turning it into new rules.

## From logs to machine learning

This is the part that connects my first internship to where I am now. Cowrie's JSON log is a clean, structured stream of attacker behaviour, and it can be turned into a dataset with very little work.

![Cowrie events are grouped into sessions, turned into features, and used for classification, clustering, sequence models, LLM triage and risk-based routing](/assets/img/posts/honeypot-lab/ml.png)

Every event carries a session ID, so the first step is grouping events into sessions: one row per visit, from `cowrie.session.connect` to `cowrie.session.closed`. Each session then has features that are hard to fake:

- timing: the gaps between login attempts, how long the session lasted, how fast commands were typed;
- credentials: which username and password pairs were tried, and how much they overlap with known wordlists;
- the client: the SSH version string, and a HASSH fingerprint of the SSH key exchange, which identifies the client software even when the IP changes;
- the commands typed after login, in order.

With that, there are a bunch of things a model could learn.

Bot or human. Most SSH attacks are scripts, and scripts are fast and regular. A human types slowly, makes mistakes and explores. Telling them apart says a lot about how worried to be.

Campaign clustering. Thousands of IPs often run the same playbook: the same wordlist, the same commands, the same file downloaded from the same server. Clustering sessions on those features groups them into campaigns, even when the IPs are all different.

Command sequences. The commands an attacker types after logging in are a sequence of actions, exactly like the API calls in my [malware detection post](/blog/2026/malware-api-calls/). The same models apply: n-grams, sequence models, transition graphs. The same trap applies too. Bots repeat the exact same session thousands of times, so a dataset of honeypot sessions will be full of duplicates, and a random train/test split will mostly test memory.

LLM triage. An analyst doesn't want to read 200 raw events. A language model can summarise a session ("logged in as root, checked the CPU, downloaded a miner, added an SSH key for persistence") and suggest the matching ATT&CK techniques. That's a direct link to the work I do now with LLMs.

Risk-based routing. In my report, I listed one limitation as "intelligent traffic management": the lab sends all port-22 traffic to the honeypot, and a smarter system would score each connection by its source, its pattern and its frequency, then decide whether it reaches a real service or the trap. That's a classification problem, and the honeypot is what generates its training data.

There's a catch, though. A honeypot only ever sees attackers, so it gives you plenty of examples of attacks and none of normal traffic. A model trained on it alone has no idea what a legitimate SSH session looks like. You need benign data from somewhere else. And attackers know about honeypots: Cowrie's defaults (like its fake hostname, `svr04`) are well known, and some bots check for them and leave. Changing those defaults is one of the first things to do in a real deployment.

## Wrapping up

This internship was my first time doing something end to end: designing a network, building it, attacking it, and watching the alerts come in. It's also where I learned that a security setup is only as good as what it lets you see. A year later, I'd fix the sensor's placement and rewrite the alert rules. But the parts that matter most, the segmentation, the single door into the trap, the logs flowing into a SIEM, still hold up.

What I find most interesting now is that a honeypot is a data generator. Every attacker who falls for it writes a few more rows of labelled data about how attacks actually happen, and that's exactly the kind of data I like to build models on.

Thanks to Yassine Ben Amor for the supervision, and to the SFM Technologies team for having me. And thanks to you for reading this far. If you run honeypots, or you think I got something wrong, I'd really like to hear about it.
