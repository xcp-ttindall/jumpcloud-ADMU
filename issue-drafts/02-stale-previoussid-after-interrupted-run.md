# An interrupted run leaves PreviousSID in the source user's own hive and blocks the next run

Status: Confirmed in code (write path and ordering). Fix on branch `fix/stale-previoussid-and-hive-unload`.

## Problem

After a run stopped before the ProfileList swap, the next run failed with:

```
[VERBOSE] [Test-PreviousSID] Found previous SID: <domainSID>. This indicates the user has been migrated before. Exiting...
[WARNING] User Profile Previous SID Error: The user profile has a PreviousSID key. The user has been migrated previously.
```

The value data was the user's own domain SID. ProfileList still mapped <domainSID> to the original folder, and there was no .bak key.

## Where ADMU writes it (Confirmed in code)

All writes are in `Start-Migration.ps1`:

1. `New-RegKey` and `Set-ValueToKey` write `previousSID` and `previousProfilePath` to `HKU\<newSID>_admu\SOFTWARE\JCADMU` (lines 1144 to 1146). That hive is loaded from the new local user's `NTUSER.DAT.BAK` (`Set-RegistryExe.ps1`, hive file for `root`).
2. `Copy-Item` copies the target `NTUSER.DAT.BAK` over the source folder's `NTUSER.DAT.BAK` (line 1192).
3. `Rename-Item` renames the source `NTUSER.DAT` to `NTUSER_original_<ts>.DAT` (line 1229), then the source `NTUSER.DAT.BAK` to `NTUSER.DAT` (line 1349).
4. ProfileList is changed only at lines 1483 and 1484.

Between steps 3 and 4 the source user's own `NTUSER.DAT` holds the merged hive, including `previousSID` equal to the source SID, while ProfileList still points at the folder. The hive files are swapped before ProfileList. A run that stops in that window leaves this state. This matches the observed file sizes (smaller merged `NTUSER.DAT`, larger `NTUSER_original_<ts>.DAT`).

The earlier run's last log line named a registry copy step, which comes before step 1 in the code. Either the run progressed past its last logged line or the log was incomplete. Cannot determine which from the code.

`previousSID` is read only by `Test-PreviousSID`. Nothing else uses it.

## Why the next run fails

`Test-PreviousSID` returns true for any non-empty value. A completed migration writes the source SID into the target hive, so the value never equals the SID of the hive owner. A value equal to the owner's SID cannot come from a completed migration.

## Fix

`Test-StalePreviousSID` treats the value as stale when it equals the owner's SID and ProfileList maps that SID to a folder without the `.ADMU` suffix. `Test-PreviousSID` then logs a warning, calls `Remove-StalePreviousSID` and returns false. The loaded hive is `NTUSER.DAT.BAK`, the copy made for the current run, so the live `NTUSER.DAT` is not edited. `Remove-StalePreviousSID` deletes the whole `JCADMU` key when nothing else is in it, because `Start-Migration` writes the new marker only when the key does not exist (lines 1132 to 1147).

Other options considered: remove the value during rollback (the hive is not loaded at that point and a killed run never reaches rollback), or write the value only after the swap (the value lives in the hive file that is swapped).

## Hives left loaded (Confirmed in code)

`Test-UserRegistryLoadState` throws from `process{}` when `Test-PreviousSID` is true. In PowerShell `end{}` does not run after a throw, so neither the unload in `process{}` nor the check in `end{}` runs. No other code unloads `<SID>_admu` or `<SID>_Classes_admu` on this path. A stub harness shows both hives still loaded with the original code. The fix wraps the validations in `try/finally` and unloads any hive still loaded.

One observation does not match the code: a later `reg query HKU` showed no hives loaded. Cannot determine what unloaded them. A manual unload, a reboot or the begin block of a later run are possible.
