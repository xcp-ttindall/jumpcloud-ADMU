# Desktop access denied on first sign-in after a migration with SetFullPermission off

Status: mechanism Confirmed in code. Cause for the single observed case Cannot determine. Seen once.

## How the deferred pass is triggered

With `SetFullPermission` false, `Start-Migration.ps1` (lines 1538 to 1578):

1. Calls `Set-RegPermission` without `-Recursive`. That grants the target SID `(OI)(CI)F` on the profile root with `icacls` and sets the owner of the root only (`Set-RegPermission.ps1`, non-recursive branch).
2. Calls `New-RegPermissionTask`, which registers `ADMU-SetPermissions-<TargetSID>`: an at-logon trigger for the new user, run as SYSTEM, `StartWhenAvailable`, 3 restarts at 1 minute, 1 hour limit. It runs `uwp_jcadmu.exe -SetPermissions 1 ...` and deletes itself on success.

`uwp_jcadmu.ps1` (permission mode, around lines 915 to 1048) uses its own `Set-RegPermission` with `icacls /grant ... /T` and then `icacls /setowner ... /T`. That is sequential and single-threaded, and the second pass starts only after the first finishes. The faster native recursive version is used only when `SetFullPermission` is true.

## Can the user open Desktop first?

Yes. The task starts at logon, in parallel with the shell. Nothing blocks the shell until the pass finishes. A folder that does not inherit the new SID's ACE, or that the source SID or Administrators still own, stays inaccessible until the `/T` pass reaches it. The tree is walked in directory order, so large folders such as AppData are processed before Desktop and Documents.

Whether the root-level grant reaches children depends on each child having inheritance enabled. Cannot determine from code whether the affected Desktop folder had it disabled.

## Options

1. Stamp the known folders (Desktop, Documents, Downloads, Pictures, Music, Videos, Favorites) synchronously during the migration with `ApplyOwnerAndGrantTree` and defer the rest. Those trees are usually small.
2. Run the deferred pass in `uwp_jcadmu` with the native recursive code instead of `icacls /T`.
3. Document it: with `SetFullPermission` off, first-login access to profile folders can fail until the scheduled task finishes. Check `jcAdmu.log` for `End Post-Migration NTFS Task`.

A manual fix that worked: take ownership and grant the new SID full control recursively on the profile.
