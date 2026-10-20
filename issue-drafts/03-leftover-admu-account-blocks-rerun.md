# A leftover ADMU account blocks a re-run, and the message blames a JumpCloud association

Status: message Confirmed in code and fixed on branch `fix/leftover-admu-user-message`. Cleanup is an enhancement request.

## State

After a run stops before the profile swap:

- A local account <user> exists with the description `Created By JumpCloud ADMU`.
- ProfileList maps <localSID> to `C:\Users\<user>.000`.
- ProfileList still maps <domainSID> to `C:\Users\<user>`.

The next run fails with `The user will not be able to be created because the device was associated to a JumpCloud user matching the same username ... User was created by JumpCloudADMU.`

## Cause

`Start-Migration.ps1` (lines 593 to 629) builds the message for the case where the account exists and `jumpCloudManaged` is false. Both ADMU-created and JumpCloud-created accounts get the "associated to a JumpCloud user" text. `Start-Reversion` does not help here. It removes ADMU accounts only when their profile path equals the reverted path (`Start-Reversion.ps1`, Remove JumpCloud ADMU Created User region), and there is no `.ADMU` path in this state.

## Message fix

`Get-LeftoverAdmuUserMessage` names the account and its profile path and picks the recovery from ProfileList:

- Before the swap: remove the account and its profile folder.
- After the swap (source path ends in `.ADMU`): the account owns the original folder, so the message says not to remove it and points to `Start-Reversion`.
- Account and source share a path: the message says not to remove the folder.

The existing Acceptance test that expected the old text is updated.

## Enhancement request: supported cleanup

Add a parameter (for example `Start-Migration -RemoveLeftoverAdmuUser`) or a mode of `Start-Reversion` that removes the stray account and its profile only after it verifies all of these:

1. The account description is `Created By JumpCloud ADMU`.
2. ProfileList maps the source SID to its original folder and the folder exists.
3. The account's profile path differs from the source profile path and is not the source folder.
4. No hive for either SID is loaded.

It should support `-DryRun` and `-Force`, and log each decision.

Related: the rollback and run-state work in `fix/interrupted-migration-recovery` reduces how often this state occurs and prints the recovery when it does.
