# Start-Reversion: `-Force` exists, but `-WhatIf` and `-Confirm` do nothing

Status: the requested feature is Not supported by code as a gap, because `-Force` exists. A separate defect is Confirmed in code.

## Unattended use

`-Force` is declared at `Start-Reversion.ps1` lines 57 and 58 and skips the prompt at line 354. It has been in the function since v2.11.0. The exe wrapper has a `Revert` parameter set that passes `Force` (`New-ADMUTemplate.ps1`). The prompt in the report appears when `-Force` is not passed. Docs/Start-Reversion.md lists `-Force`.

## Defect

The function declares `[CmdletBinding(SupportsShouldProcess)]` (line 41) and never calls `$PSCmdlet.ShouldProcess`. As a result:

- `Start-Reversion -WhatIf` performs the reversion. `-DryRun` is the only dry-run switch.
- `-Confirm:$false` is accepted and has no effect. The `Read-Host` prompt still appears.

## Options

1. Remove `SupportsShouldProcess` so `-WhatIf` and `-Confirm` are rejected.
2. Wire it in: map `-WhatIf` to the `DryRun` behavior and treat `-Confirm:$false` like `-Force`.

Option 2 matches PowerShell conventions. This is a maintainer decision, so no code change is proposed here.
