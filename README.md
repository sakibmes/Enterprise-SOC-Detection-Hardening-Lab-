# Enterprise-SOC-Detection-Hardening-Lab-

Enterprise SOC Lab: Adversary Simulation to Detection Engineering

Adversary simulation, SIEM detection engineering, MITRE ATT&CK mapping, and CIS hardening with before/after proof

Why I Built This

I wanted to stand up a realistic enterprise environment, attack it like an adversary would, catch those attacks in a SIEM, harden the environment, and then re-run the same attacks to prove the fixes actually worked.

The full defensive lifecycle, with evidence at every step. This is that project.

Hardware and Environment

My hardware is a single Dell OptiPlex 3046 with 16GB of RAM running Proxmox. That cap drove a real engineering decision: I chose Wazuh over Splunk as my SIEM, because Splunk's footprint would've eaten my RAM budget with three other VMs running.

Working within constraints is the job, so I leaned into it.

The environment:

Wazuh SIEM, the detection system
Windows Server 2022 Domain Controller, corp.local, with users, OUs (IT / HR / Finance), and groups provisioned through PowerShell. Seeded with realistic accounts plus one weak-password target (bweak) and a service account (svc_sql).
Kali Linux, the attacker
Sysmon (SwiftOnSecurity config) on the DC, feeding rich endpoint telemetry into Wazuh
Repo Structure
architecture/
attacks/
detections/
hardening/
investigations/
mitre/
The Attack Chain

I worked through a realistic kill chain, watching what the SIEM caught at each stage.

1. Recon

A basic nmap -A scan fingerprinted the DC: Kerberos (88), LDAP (389), SMB (445), RDP (3389). Classic domain controller exposure.

2. Password Spraying

Using NetExec against SMB, I cracked bweak:Password1. Wazuh lit up: 50 failed logons, a correlation alert for "Multiple Windows Logon Failures," and an automatic mapping to MITRE ATT&CK Brute Force (T1110). This is detection engineering working as designed.

3. Brute-Force and Two Real Findings

I ran Hydra against RDP and it failed, not because the password was wrong, but because bweak had no RDP rights. That's a least-privilege finding. Hydra also failed against SMB, which turned out to be a legacy-tooling issue against modern SMBv2/3, a tooling finding. I tried NetExec and got a clean authentication.

4. Enumeration and the Detection Gap That Taught Me the Most

Using LDAP queries, I enumerated every domain user (with their bad-password counts) and every group, mapping out exactly who held privilege. From an attacker's view, it worked perfectly.

But here's the lesson: Wazuh only caught it as a vague, level-3 "Discovery activity." No real severity, no specificity. That's a genuine blind spot, the kind of quiet recon a SOC can easily miss. I also noticed LDAP signing was set to None, which opens the door to relay attacks.

5. Kerberoasting: Attempted, and Documented Honestly

I attempted a Kerberoasting attack against the service account. It failed, not on defenses, but on a persistent VM clock-skew issue (KRB_AP_ERR_SKEW) that kept drifting even after NTP sync.

I'm documenting it as a lab limitation rather than dressing it up. Worth noting: the attempts themselves threw severe flags in Wazuh, so even the failed attack was a detection win.

The Defense and the Proof

Detection is half the job. The other half is fixing what you found.

I ran a CIS Benchmark baseline scan against the DC first: 27% (99 of 359 checks passing). That's my "before."

Then I hardened via Group Policy and learned a quirk the hard way: account lockout and password policies have to live in the Default Domain Policy, not a custom GPO, or they silently don't apply. I diagnosed that after my first settings didn't take.

I set an account lockout threshold of 5, a 14-character minimum password length, enabled complexity, and enforced LDAP signing.

I Re-Ran the Attack

I pointed the same password spray back at bweak. This time the account locked after 5 attempts, and even when the tool tried the correct password, it returned STATUS_ACCOUNT_LOCKED_OUT. The attack that worked an hour earlier was now dead. And Wazuh caught the lockout in real time (Event ID 4740).

The CIS checks for password policy and lockout flipped from FAIL to PASS.

What I Actually Learned
Detection isn't binary. Wazuh caught my loud attacks easily but nearly missed my quiet enumeration.
Hardening means nothing without validation. Re-attacking is the step most people skip, and it's the only one that actually proves anything.
