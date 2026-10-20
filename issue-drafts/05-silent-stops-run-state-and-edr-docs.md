# Runs that end with nothing logged: add a run-state marker, step logging and EDR guidance

Status: cause Cannot determine. Detectability changes are on branches `feat/run-state-marker` and `feat/log-registry-step-exit-codes`. Docs text below is for the wiki.

## Observation

Several runs started from a JumpCloud command (SYSTEM) ended with no ERROR or WARN line. The last lines were in the registry steps, for example `====== Copying User Registry Source to Target ======`, or after the hive rename. They ended between 48 seconds and 4 minutes after "Migration Start". The same users migrated through the GUI at the console. An EDR agent was present. No detection has been confirmed, and the JumpCloud command timeout is unconfirmed. This draft states no cause.

## What the code shows

- `Write-ToLog` appends each line immediately (`Add-Content`), so a killed process loses no lines. A silent end means the process stopped between log lines.
- `end{}` and the rollback in it do not run when the process is killed or when `process{}` throws. A killed run therefore also leaves no revert, which is the source of several follow-on states (drafts 02, 03 and 08).

## Changes

1. Step logging (`feat/log-registry-step-exit-codes`): log before and after each `reg copy` (both hives, first attempt and retry) with the exit code, log the `takeown` exit code and log around the hive backup copy. The `$?` checks are preserved.
2. Run-state marker (`feat/run-state-marker`): `HKLM:\SOFTWARE\JCADMU\RunState` is written after validation, set to phase `swapped` after the ProfileList update and cleared in `end{}` and before a forced reboot. On launch, an existing marker is logged with the recovery for its phase, or a note that the recorded process is still running. Validation failures do not leave a marker.

## Proposed wiki text for the Limitations page

> ### Endpoint protection and non-interactive launches
>
> ADMU loads and unloads user registry hives, copies and renames NTUSER.DAT and UsrClass.dat, and sets ownership on profile folders. Endpoint detection and response (EDR) products can block or stop these operations, especially when ADMU is started by a service account rather than at the console. A blocked run can end without an error in `C:\Windows\Temp\jcAdmu.log`.
>
> - Exclude `gui_jcadmu.exe` and `uwp_jcadmu.exe` by publisher certificate in your EDR policy. Where that is not possible, pause the agent on the device for the migration window.
> - When you run ADMU from a JumpCloud command, set the command timeout longer than the longest expected migration. Large profiles take much longer than small ones.
> - After an unexplained stop, read the last log line, then follow the recovery in the log of the next run.

## Needs a maintainer decision

Whether the marker should also survive a handled failure (it is cleared in `end{}` today).
