# Authentication and Authorization — Technical Reference

This reference explains the recorded access model and conventional checks. Use the [manual guide](GUIDE.md) for the six-step walkthrough.

## Permission model

`CyberAndChill.com` authenticates two standard accounts on the joined `CC-WIN11` client. Both accounts can sign in. The tested resource is one SMB share, `\\DC01\CCIAM`, mapping to `C:\CyberChillIAM\Protected` on DC01.

| Layer | Principal | Grant |
|---|---|---|
| SMB share | Authenticated Users | Read |
| Protected NTFS folder | SYSTEM | Full control, inherited by child content |
| Protected NTFS folder | Administrators | Full control, inherited by child content |
| Protected NTFS folder | GG_IAM_Readers | Read and execute, inherited by child content |

The folder's DACL has inheritance disabled and contains these three allow entries. Only `iam-reader` belongs to `GG_IAM_Readers`. There is no direct allow for either training account and no explicit deny for the nonmember.

An SMB operation must pass both share permissions and NTFS permissions. Here both users satisfy the share's authenticated-read condition, while only the group member satisfies the folder's NTFS read condition. The reader lacks write rights at both layers; its denied write-open does not isolate which layer rejected the operation first.

The SYSTEM/Administrators entries preserve local administrative management. The share itself remains read-only, including for an administrator using that share. This controlled training exercise reuses the existing controller; a production file-service design should use the organization's designated file server and policies.

## Conventional inspection commands

Run these on the appropriate lab machine, using authorized administrative access only where required. They inspect the existing configuration; they do not create accounts, reset passwords or change permissions.

On the client, as each standard user:

```bat
whoami
whoami /groups
echo %LOGONSERVER%
net group GG_IAM_Readers /domain
icacls \\DC01\CCIAM
type \\DC01\CCIAM\welcome.txt
```

`net group` checks directory membership. `whoami /groups` checks the current process token; these are distinct observations. The nonmember can receive Access Denied from `icacls` as well as from the content read. Use the server to inspect permissions independently of that user's denied access.

On DC01, open an authorized PowerShell session:

```powershell
Get-ADGroup GG_IAM_Readers -Properties GroupScope, GroupCategory
Get-ADGroupMember GG_IAM_Readers
Get-SmbShare CCIAM | Select-Object Name, Path
Get-SmbShareAccess CCIAM
(Get-Acl 'C:\CyberChillIAM\Protected').Access
```

These use the server's existing Active Directory and SMB cmdlets. No companion setup script or external service is required to follow the guide.

## Recorded verification and limits

The [public results](evidence/results.json) contain selected measurements from the recorded run. In addition to the native desktop tests, an interactive-logon API created a fresh type-2 token for each account. Under each identity, a real SMB file read was attempted against the server. The reader succeeded; the nonmember received `UnauthorizedAccessException`. The reader's attempt to open the existing file for writing was denied without modifying its content.

The Windows screenshots show the actual identities, logon controller, group/ACL results, file content and Explorer denial. They were produced by automated GUI interaction. Configuration used automation; the written GUI configuration sequence is a conventional equivalent derived from the recorded settings, rather than a claim of independent human manual execution.

A controlled controller restart occurred during verification. The share, group membership and protected folder ACL persisted; directory connectivity was re-established and the token/SMB checks passed afterward. This does not establish client-restart behavior or every application's recovery behavior. The publication process did not rerun the lab.

Host capture times are recorded in UTC in [image provenance](evidence/image-provenance.json). Early client UTC values differed from that clock; the images and original measurements were not altered to hide that difference. No Kerberos-only or cloud-authentication claim is made.

All visible identities, domain names, machine names and directory identifiers belong to this synthetic Cyber and Chill lab. Separate publication copies retain useful native interface context; no redactions were required. Credentials, private sign-in dialogs, unrelated environments and private implementation are excluded.

## Troubleshooting

- **Neither user can sign in:** Confirm account enablement, password validity, domain DNS, time alignment and trust. A sign-in failure is not evidence that the folder ACL works.
- **Both users can open the folder:** Check the actual signed-in identity, group nesting, broad inherited NTFS entries, share target and saved alternate SMB credentials. Do not add an explicit deny before understanding the unintended allow.
- **Neither can open the folder:** Verify that the server is available, the share targets the intended folder, the reader's token contains the group and both permission layers permit reading. Distinguish a network-path error from Access Denied.
- **Membership differs from token:** Sign out and back in, then inspect `whoami /groups`. Close sessions using alternate credentials. Do not infer fresh authorization merely from a changed directory membership listing.
- **A name already exists:** Check ownership, contents and dependencies. Preserve unrelated objects; choose new dedicated training names rather than repurposing them.

The retained lab remains usable for teaching. Cleanup is a separate deliberate operation: remove only verified training objects after reviewing dependencies. No cleanup was performed for publication.
