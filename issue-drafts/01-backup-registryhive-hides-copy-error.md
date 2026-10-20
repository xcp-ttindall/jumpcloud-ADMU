# Backup-RegistryHive hides the copy error and never retries

Status: Confirmed in code. Fix on branch `fix/backup-registryhive-error-logging`.

## Problem

When the first hive copy fails, the log shows `Cannot validate argument on parameter 'Message'. The argument is null or empty.` instead of the copy error, and the run stops.

```
[VERBOSE] [Backup-RegistryHive] Initial backup was not successful, trying again...
[VERBOSE] [Backup-RegistryHive] Could Not Backup Registry Hives in C:\Users\<user>: Exiting...
[WARNING] Registry Backup Error: Could not take a backup of the user registry files. ...
[VERBOSE] [Backup-RegistryHive] Cannot validate argument on parameter 'Message'. The argument is null or empty. ...
```

## Cause

`Backup-RegistryHive.ps1` (catch block) calls `Write-ToLog $CloseResults -Level Verbose`. `$CloseResults` is assigned only in a commented-out `Close-ProcessByOwner` line, so it is null. `Write-ToLog` declares `-Message` with `[ValidateNotNullOrEmpty()]` (`Write-ToLog.ps1`, parameter block), so the call throws. The throw sits inside the retry `try`, before `Start-Sleep` and before the second `Copy-Item`. The inner catch then logs the validation error as the exception text.

Effects:

1. The original exception text, source path and destination path are never logged.
2. The retry copy never runs, so every first-attempt failure, including a transient lock, ends the run.

## Reproduction

A stub harness that fails `Copy-Item` once and then succeeds shows 1 copy call and the validation error with the original code. With the fix it shows 3 calls (1 failed, 2 on retry) and no error.

## Fix

Remove the dead log call. Log the exception text, source path and destination path on the first attempt and on the retry.

## Not determined

The failing copy took about 26 seconds on one device and about 70 seconds on a device with a 100 GB profile. The code does not explain the delay. The new log lines show which copy failed and with what error.
