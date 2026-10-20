# 3_ADMU_Invoke.ps1: quser noise, TrimStart misuse, and logoff order

Status: both defects Confirmed in code and fixed on branch `fix/invoke-script-quser-and-domain-status`. The ordering claim is Not supported by code.

## quser stderr

`$loggedInUsers = (quser) -replace ...` had no stderr redirect. With no session, `quser` writes `No User exists for *` to stderr, and a JumpCloud command shows it after the `Failed Users` list as if it were a failure. Fix: `Get-LoggedInUserSession` runs `quser 2>$null`, returns nothing when there is no output, and the script logs `No signed-in users to log off.`

## TrimStart

`Get-DomainStatus` used `$line.TrimStart('AzureADJoined : ')` and `TrimStart('DomainJoined : ')`. `TrimStart` removes any leading characters in the set, not a prefix. The code works only because `YES` and `NO` start with letters outside the set. A value such as `ndj` returns `j`. Fix: a regex capture, `^\s*AzureADJoined\s*:\s*(\S+)`. The same function exists in `Powershell/Private/SystemInfo/Get-DomainStatus.ps1` and is fixed too. `Form.ps1` (lines 477 to 486) uses the same pattern for `AzureADJoined`, `WorkplaceJoined`, `WorkplaceTenantName` and `DomainJoined`. It displays tenant names, so a tenant name that starts with letters from the label text is shown trimmed. That is not changed in the branch.

## Logoff order

A note said the logoff block runs before CSV and exe validation. In this version it does not. The script validates parameters, imports users and checks the exe, and only then logs users off (`#region dataImport`, `#region exeValidation`, `#region logoffUsers`). The logoff does happen before ADMU's own per-user validation, so a user can be logged off for a run that then fails validation. Consider moving the logoff into the per-user loop after validation.

## Other notes

- The script ships placeholders for `$JumpCloudAPIKey` and `$JumpCloudOrgID`. Keys pasted into a command or chat should be rotated.
- The script is about 40,000 characters. The test that checks the 32,767 limit is skipped.
