# Leaving the domain strands the other domain profiles on the device

Status: Confirmed in code (behavior). Needs a design decision.

## Observation

On a device with other domain users who are not being migrated (for example other staff, an admin account, a former user), ADMU migrates one user and, with `LeaveDomain` or `AutoBindJCUser`, unjoins the device (`Start-Migration.ps1`, leaveDomain region; `AutoBindJCUser` forces `LeaveDomain` at line 446). The other domain profiles stay on disk, but their accounts can no longer authenticate, so those users cannot sign in.

## Requests

1. Log a warning before the unjoin that lists the other domain profiles in ProfileList that are not migrated in this run.
2. Document the behavior and a supported order: migrate every domain user first, or remove profiles that are not needed.
3. Optionally add a parameter that stops before the unjoin when other domain profiles exist.

`3_ADMU_Invoke.ps1` already defers `LeaveDomain` to the last user in a batch (`$LeaveDomainAfterMigration`), which helps for batches but not for users who are not in the CSV.
