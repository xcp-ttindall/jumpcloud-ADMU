# ADMU issue drafts

Drafts for upstream issues against ADMU v2.16.3. All identifiers are placeholders (<user>, <domainSID>, <localSID>, DOMAIN, <device>). Check each draft before posting.

Status labels: Confirmed in code, Not supported by code, Cannot determine.

Duplicate search was not possible from this environment. Search the upstream issue list for these terms before filing:

- `Backup-RegistryHive`, `Cannot validate argument on parameter 'Message'`
- `PreviousSID`, `Test-PreviousSID`, `migrated previously`
- `Created By JumpCloud ADMU`, `.000`, `will not be able to be created`
- `Start-Reversion`, `-Force`
- `silent`, `no error`, `Copying User Registry Source to Target`
- `locked by the System Kernel`, `REG UNLOAD`
- `Access denied` `Desktop`, `SetFullPermission`
- `No User exists for *`, `quser`, `TrimStart`
- `leave domain`, `domain profiles`

Each fix draft names the branch on the fork that holds the change. Tests were written in the repo's Pester layout but not run (they need Windows and Pester, which were unavailable). Logic was checked with stubbed PowerShell 7.4 scripts.
