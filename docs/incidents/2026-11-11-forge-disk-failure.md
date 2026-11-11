# Incident - forge disk failed SMART extended test

**Date:** 2026-11-11  
**Host:** forge (Debian 13 laptop)  
**Impact:** None to services. It was found before anything important was stored on the disk.

## Summary
A routine health check of a reused 500 GB laptop HDD found a failing disk, even though the overall SMART status said "PASSED". The host was reclassified as disposable lab hardware.

## Timeline
1. After installing Debian, installed `smartmontools` and ran `smartctl -a`.
2. The overall health said PASSED, but the attributes and the error log showed problems:
   - `Reported_Uncorrect`: 10
   - `Command_Timeout` raw value: 54
   - ATA error log: 9 errors, at about 5,168 power-on hours
3. Short self-test (`smartctl -t short`): passed.
4. Extended self-test (`smartctl -t long`, about 140 min): **Completed: read failure** at 90% remaining, with one failing LBA.
5. After the test:
   - `Current_Pending_Sector` 0 → 1
   - `Offline_Uncorrectable` 0 → 1
   - `Reported_Uncorrect` 10 → 11

6. Recheck about 4 hours later (idle): every error counter was unchanged. The fault is present but not getting worse at rest. A side finding: `Load_Cycle_Count` rose by about 100 per hour from the laptop drive's aggressive head parking, which would exceed its rated lifetime in under a year of 24/7 use.

## Analysis
- The overall SMART result only fails when an attribute crosses the manufacturer's threshold. It is a lagging indicator.
- A failed extended test plus a pending (unreadable) sector is direct evidence that the media can't be read reliably.
- `Reallocated_Sector_Ct` = 0 means the drive hasn't remapped the bad sector yet. It will try only when that sector is next written.

## Decision
- Keep using forge for disposable learning only.
- No destructive repair (`badblocks -w`, forced sector rewrites). It would hide evidence and wouldn't make the disk trustworthy.
- If forge becomes useful long-term, replace the drive with a cheap 2.5" SATA SSD.

## Lessons
- Don't trust "SMART overall-health: PASSED" on its own. Read the attributes and the error log, and run the extended test on any reused disk before relying on it.
- The same check is next for the Proxmox host's 2 TB disk.
