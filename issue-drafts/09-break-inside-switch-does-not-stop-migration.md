# A failed registry copy does not stop the migration

Status: Confirmed in code and by a PowerShell 7.4 check. Not fixed on a branch.

## Problem

In `Start-Migration.ps1`, the retry of `reg copy` is checked with `switch ($?)` (lines 1007 to 1017 and 1059 to 1068). The `$false` case sets `$admuTracker.copyRegistry.fail = $true` and calls `break`. In PowerShell, `break` inside `switch` leaves the `switch`, not the enclosing `while`. A short script confirms it: the statement after the `switch` still runs.

Effects: after both copy attempts fail, the run continues. Line 1090 then sets `$admuTracker.copyRegistry.pass = $true`, so the tracker reports a step as both failed and passed, and the run proceeds toward the swap with an unmerged target hive.

## Fix options

Use a flag and `break` after the `switch`, or replace the `switch` with `if/else`. The same pattern should be checked for other `switch` blocks that use `break` to leave the migration loop.
