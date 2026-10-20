# A failed or interrupted run leaves the user's profile broken instead of restored

Status: gaps Confirmed in code. Fix for failures that reach `end{}` is on branch `fix/interrupted-migration-recovery`. Cause of the hive unload failures Cannot determine.

## Observed states

Across several devices, failed runs left users unable to sign in:

- A run died after the hive files were swapped but before ProfileList was updated. The user then had a merged `NTUSER.DAT`, `NTUSER_original_<ts>.DAT` next to it, and a stray local account.
- A run died after the swap: <domainSID>.bak pointed at a missing `C:\Users\<user>.ADMU`, the main key pointed at a TEMP profile, and the ADMU account owned `C:\Users\<user>`.
- A run failed at `REG UNLOAD` of the new profile hive after 5 retries, with `locked by the System Kernel`. The rollback removed the account but not the files. The profile folder removal then failed with `The directory is not empty`. The copy step had already overwritten the source `UsrClass.dat`.
- After a restart the user saw a black screen with a failure message at sign-in. The User Profile Service events were clean and the profile loaded.

## What the code does

1. The `end{}` rollback handles only `newUserInit` (`Start-Migration.ps1`, end block). It removes the new account and profile and re-enables scheduled tasks. It does not unload hives, restore `NTUSER.DAT` or `UsrClass.dat`, or reverse the swap.
2. `copyRegistryFiles` copies the new user's hive backups over the source folder's backups (line 1192). `renameOriginalFiles` and `renameBackupFiles` then replace the live hives. After these steps the source user's live hives are the merged ones.
3. The merged hive contains `Software\JCADMU`. `uwp_jcadmu.ps1` runs its first-login app when `HKCU:\SOFTWARE\JCADMU` exists (line 1062). A user who signs in with a merged hive from a failed run therefore starts the migration UWP app, which expects the rest of the migration to have happened. This is a plausible cause of the black screen. Cannot determine for the observed case.
4. The message `locked by the System Kernel` is printed whenever Restart Manager lists no locking process (`Set-RegistryExe.ps1`). It is not evidence of a kernel lock.
5. `end{}` does not run when the process is killed or when `process{}` throws. No rollback runs in those cases.

## Fix on the branch

Before the ProfileList swap, `end{}` now:

- unloads the source and target hives if still loaded (`Clear-LoadedUserRegistryHive`),
- restores the original `NTUSER.DAT` and `UsrClass.dat` from the `*_original_<ts>` files and keeps the merged files as `*_migrated`, like `Start-Reversion` (`Restore-SourceUserRegistryFile`),
- then removes the new account and profile as before.

The run-state marker (draft 05) prints the recovery when a run was killed.

## Remaining work

- Failures after the swap still need an automatic `Start-Reversion` call, or a documented decision not to.
- The unload failures need the cause found. The EDR role is unconfirmed. Check the EDR console for blocks of `gui_jcadmu.exe` and `reg.exe` at the failure times.
- `Backup-RegistryHive` hangs of 26 to 70 seconds (draft 01) are unexplained.
- `Remove-LocalUserProfile` should report a failed folder removal as a warning with the path.
