# Just-in-Time Workstation Administration — Technical Reference

This reference complements the [GUI guide](GUIDE.md). All identifiers, names and DNs in examples are documentation values. Configuration was applied through supported interfaces; native GUI navigation and Windows console behavior were tested through automated interaction. A full independent manual rebuild was not performed.

## Starting environment and scope

Reuse a working midPoint 4.10.5 native repository and AD LDAP resource, a trusted LDAPS certificate, an existing synthetic standard IT user, an existing distinct reviewer, and one Windows 11 Pro domain member. WindowsServer 2022 supplies AD. No additional PAM, approval, recommendation or scheduling product is required.

The group grants local administration only where explicitly added to local Administrators. The role alone cannot enforce workstation scope if someone later nests or deploys the group elsewhere. Preserve that constraint and audit the group's use. Directory service identity permissions remain restricted to the owned training user/group containers; no Domain Admin grant is needed.

The existing integration must support **native group associationFromLink** from user account to group entitlement and removal of no-longer-desired memberships. Verify strict association tolerance is appropriate for the **already-isolated** synthetic scope. Do not change global tolerance across an unrelated directory to reproduce this exercise. Retain existing application memberships and the user's base account role.

## Conventional local group command

Use **Computer Management → Local Users and Groups → Groups → Administrators → Add** on the target workstation. If a conventional CLI is preferred, run the native cmdlet in an elevated 64-bit PowerShell session on **that workstation only**:

```powershell
Get-LocalGroupMember -Group Administrators
Add-LocalGroupMember -Group Administrators -Member 'LAB\PAM-Workstation-Admins'
Get-LocalGroupMember -Group Administrators
```

These are standard administration commands, not a PowerShell expiration service. The recorded build used the standard cmdlet after `net.exe` returned a syntax error. Verify actual group identity before any retry. Do not remove/recreate all Administrators members.

## Native group projection and access role

The following XML is an **adaptation template**, not the deployed object or a universal resource configuration. Substitute your existing AD resource OID and real lab DN. The existing resource must already define `entitlement/group` and the native association mapping.

In **Configuration → Import object**, import a new group role only if no owned equivalent exists:

```xml
<role xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      xmlns:ri="http://midpoint.evolveum.com/xml/ns/public/resource/instance-3"
      oid="11111111-1111-4111-8111-111111111111">
  <name>PAM Workstation Group</name>
  <description>Lab-owned group for one test workstation</description>
  <assignment>
    <construction>
      <resourceRef oid="22222222-2222-4222-8222-222222222222"/>
      <kind>entitlement</kind><intent>group</intent>
      <attribute><ref>ri:dn</ref><outbound><strength>strong</strength>
        <expression><value>CN=PAM-Workstation-Admins,OU=Groups,OU=Training,DC=ad,DC=lab,DC=example,DC=com</value></expression>
      </outbound></attribute>
      <attribute><ref>ri:cn</ref><outbound><strength>strong</strength>
        <expression><value>PAM-Workstation-Admins</value></expression>
      </outbound></attribute>
      <attribute><ref>ri:sAMAccountName</ref><outbound><strength>strong</strength>
        <expression><value>PAM-Workstation-Admins</value></expression>
      </outbound></attribute>
      <attribute><ref>ri:groupType</ref><outbound><strength>strong</strength>
        <expression><value>-2147483646</value></expression>
      </outbound></attribute>
    </construction>
  </assignment>
</role>
```

Verify the actual group and the role's projection before continuing. If you created the group in AD, link the exact existing entitlement shadow to the role rather than provisioning another object. The access role induces that group role:

```xml
<role xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      xmlns:c="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      oid="33333333-3333-4333-8333-333333333333">
  <name>Temporary Workstation Admin</name>
  <description>Temporary administrator on one test workstation</description>
  <requestable>true</requestable>
  <inducement>
    <targetRef oid="11111111-1111-4111-8111-111111111111" type="c:RoleType"/>
  </inducement>
</role>
```

Do not copy the reference OIDs into a working environment without checking collisions. The example user OID below is `44444444-4444-4444-8444-444444444444`; the existing reviewer is `55555555-5555-4555-8555-555555555555`.

## Native approval rule

In **Configuration → Repository objects → System configuration**, retain the existing object and append a new `globalPolicyRule` with the following shape. Declare `c` as the common namespace and `q` as `http://prism.evolveum.com/xml/ns/public/query-3` on the containing object. Do **not** replace the complete system configuration or existing rules.

```xml
<globalPolicyRule>
  <name>Single workstation PAM approval</name>
  <policyConstraints><assignment><operation>add</operation><scope>any</scope></assignment></policyConstraints>
  <policyActions><approval>
    <compositionStrategy><order>10</order></compositionStrategy>
    <approvalSchema><stage>
      <name>Temporary workstation access review</name>
      <approverRef oid="55555555-5555-4555-8555-555555555555" type="c:UserType"/>
      <evaluationStrategy>firstDecides</evaluationStrategy>
      <outcomeIfNoApprovers>reject</outcomeIfNoApprovers>
    </stage></approvalSchema>
  </approval></policyActions>
  <evaluationTarget>assignment</evaluationTarget>
  <focusSelector><type>c:UserType</type><filter><q:inOid>
    <q:value>44444444-4444-4444-8444-444444444444</q:value>
  </q:inOid></filter></focusSelector>
  <targetSelector><type>c:RoleType</type><filter><q:inOid>
    <q:value>33333333-3333-4333-8333-333333333333</q:value>
  </q:inOid></filter></targetSelector>
</globalPolicyRule>
```

The reviewer must have existing case read/completion authority. A reviewer configured only for REST does not automatically have the self-service GUI authorizations needed for a conventional browser review. For that alternative, configure the built-in **Approver** role and required GUI access; that full reviewer GUI decision path was not independently replayed in this test. The recorded requester was the setup administrator. An existing onboarding service's request allowlist remained unchanged; do not claim the standard Windows user requested its own access through self-service.

Verify a real pending case and absence of target membership before deciding. Requester-service case-completion denial and a distinct reviewer decision were actually tested. If approval is unavailable, disclose an explicitly authorized direct assignment instead of calling it approval.

## Validity and reconciliation tasks

The native scanner checks validity boundaries and recomputes the object. [Evolveum's 4.10 documentation](https://docs.evolveum.com/midpoint/reference/support-4.10/tasks/specific/focus-validity-scan/) explains the last-scan interval and limitations for removing multi-value associations. The final expiry-and-reconciliation task supplies native removal plus a repeated target check/retry in addition to boundary scanning. Both tasks are scoped to one user; the existing global schedules are preserved.

Import the scanner below and the final expiry-and-reconciliation task in [Expired-assignment processing](#expired-assignment-processing-and-the-failed-first-trial) through **Configuration → Import object**, after replacing the example user OID. Inspect **Server tasks → All tasks** to confirm runnable state and repeated successful completion. The schema uses seconds for these interval values.

```xml
<task xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      xmlns:c="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3">
  <name>PAM workstation validity scan</name>
  <ownerRef oid="00000000-0000-0000-0000-000000000002" type="c:UserType"/>
  <executionState>runnable</executionState>
  <schedule><interval>30</interval></schedule>
  <activity><work><focusValidityScan><objects>
    <type>c:UserType</type><query><q:filter><q:inOid>
      <q:value>44444444-4444-4444-8444-444444444444</q:value>
    </q:inOid></q:filter></query>
  </objects></focusValidityScan></work></activity>
</task>
```



[Native recomputation work options](https://docs.evolveum.com/midpoint/reference/support-4.10/tasks/activities/work/) support reconcile. This intentionally revisits the test user's desired memberships; do not broaden its query to an entire population.

## Assignment window

Set validity on the **assignment**, not on the user. Disabling the entire user at expiry would remove ordinary sign-in as well as administrative access and would not demonstrate the intended result. A safe example 15-minute interval is:

```xml
<assignment xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
            xmlns:c="http://midpoint.evolveum.com/xml/ns/public/common/common-3">
  <targetRef oid="33333333-3333-4333-8333-333333333333" type="c:RoleType"/>
  <activation>
    <validFrom>2030-01-01 T 12:00:00 Z</validFrom>
    <validTo>2030-01-01 T 12:15:00 Z</validTo>
  </activation>
</assignment>
```

These are **example future timestamps**, not measured evidence. Use your real intended window. Approval may complete after validFrom, shortening the actual period of granted membership. Record the membership timestamps separately from the configured 900-second interval.

## Windows enforcement and token boundaries

Use `fsutil dirty query C:` only inside the lab workstation. [Microsoft documents the query](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/fsutil-dirty); never use its `set` operation for this exercise. A successful directory write is insufficient: the actual read-only query must fail before elevation, succeed under the same user's elevated context, and fail after expiration plus fresh sign-in.

[Windows access tokens](https://learn.microsoft.com/en-us/windows/win 32/secauthz/access-tokens) carry a security context created during logon. Removing AD membership does not terminate an existing elevated process. UAC remains enabled; an unelevated process can be denied even while the user is eligible to elevate. Conversely an old elevated process can continue succeeding after group removal. Keep these three observations distinct:

| Observation | What it establishes |
|---|---|
| AD group membership removed | The identity no longer has the directory entitlement |
| Existing elevated process still succeeds | Its previously issued token remains usable |
| Same user denied after full sign-out/sign-in | A fresh Windows session no longer receives administrator capability |

Lock/unlock and opening another tab in an old elevated terminal are not fresh sign-ins. Keep AD reachable to avoid testing cached offline membership. Never enter an independent administrator's credentials and attribute that success to the standard user.

## Timing, recovery and limits

Read [measured results](evidence/results.json) for the final expiry observation bounds and tested recovery. Samples every 5 seconds establish an observation interval, not an exact directory commit time. Distinguish schedule cadence, assignment validity and actual Windows-session duration. No 15-minute hard cutoff, forced session termination, production SLA, host reboot recovery or outage spanning the expiry boundary is claimed.

The bounded recovery test restarts only the existing identity-management application during active access, then checks that both retained native tasks complete again before observing expiry. Database and configuration persist. The observer is read-only and does not trigger revocation. This tests application restart/task resumption; it does not prove recovery under every failure mode.

## Troubleshooting

- **Membership appears before approval:** Inspect the exact policy target selector and pending case; stop the test and correct scope before claiming approval. Do not broaden the role catalog or reviewer rights.
- **Approval succeeds but membership is initially absent:** Inspect the execution case/task and target result. Asynchronous completion may lag the decision; wait and observe rather than resubmitting.
- **Membership stays after validTo:** Confirm correct timezone, runnable tasks, last successful completion, exact user scope, AD connectivity and native association removal behavior. Inspect the first failure. Reconciliation must verify actual target membership, not only a shadow.
- **Fresh user can still elevate:** Verify full sign-out, reachable DC, nested groups, other local administrator grants, cached offline logon and use of the intended identity. Do not disable UAC or reset established accounts.
- **Old elevated window still works:** Record it as the token limitation, then complete a fresh-sign-in test. Membership revocation is not session termination.
- **Role assignment remains listed:** Inspect effective activation and AD membership. Expiration need not delete the historical assignment record.
- **HTTP 202 during task creation:** This is asynchronous acceptance. Inspect the existing task before repeating the request.
- **Application startup connection refused:** Check retained container health and readiness; do not initialize the database or recreate volumes.

## Safe cleanup

After verifying the expired state, retain the owned objects for inspection if desired; expired access is not active access. To dismantle this lab, first verify no active PAM assignment/member remains, suspend only the two dedicated tasks, remove only the owned access-role assignment/policy, unlink the dedicated group from the single workstation, then remove the owned group-role projection and access role after dependency inspection. Preserve all baseline memberships, service identities, prior project roles and the independent administrator. Do not run generic reset or teardown instructions against a shared directory.

**Clock qualification:** The client initially used Pacific time and its UTC clock was three hours ahead. Earlier Windows screenshot clock labels are not authoritative event timestamps. Capture provenance and the membership timeline use the aligned management/directory clocks. Domain time synchronization was corrected before the final fresh-sign-in denial check; the client timezone itself was preserved.


## Expired-assignment processing and the failed first trial

The first complete 15-minute trial failed: the assignment became disabled, but AD retained membership despite successful validity and recomputation tasks. Mapping-range experiments did not establish reliable removal and were reverted. Those trials are excluded from the final passing result.

The final implementation keeps the proven shared association mapping unchanged and replaces the recomputation-only job with a supported native **iterative scripting task**. Its built-in `unassign` action removes **only the disabled PAM assignment on the exact test user**, then native recompute with reconciliation verifies desired target state. There is no external removal script, PowerShell expiration workflow, custom scheduler or date-coded cleanup job.

Use the following final task alongside the validity scanner. Suspend the obsolete test task so it does not run in parallel. Keep the 30-second validity scanner: it derives the assignment's disabled effective status from validTo. The subsequent task runs every60seconds. Filters must include both the PAM role OID and disabled activation; omitting either changes the scope.

```xml
<task xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      xmlns:c="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
      xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3"
      xmlns:s="http://midpoint.evolveum.com/xml/ns/public/model/scripting-3">
  <name>PAM expiry and reconciliation</name>
  <ownerRef oid="00000000-0000-0000-0000-000000000002" type="c:UserType"/>
  <executionState>runnable</executionState>
  <schedule><interval>60</interval></schedule>
  <activity><work><iterativeScripting>
    <objects><type>c:UserType</type><query><q:filter><q:inOid>
      <q:value>44444444-4444-4444-8444-444444444444</q:value>
    </q:inOid></q:filter></query></objects>
    <scriptExecutionRequest>
      <s:pipeline>
        <s:unassign><s:filter><q:and>
          <q:ref><q:path>targetRef</q:path>
            <q:value oid="33333333-3333-4333-8333-333333333333" type="c:RoleType"/>
          </q:ref>
          <q:equal><q:path>activation/effectiveStatus</q:path><q:value>disabled</q:value></q:equal>
        </q:and></s:filter></s:unassign>
        <s:action><s:type>recompute</s:type>
          <s:parameter><s:name>reconcile</s:name><s:value>true</s:value></s:parameter>
        </s:action>
      </s:pipeline>
    </scriptExecutionRequest>
  </iterativeScripting></work></activity>
</task>
```

This uses [midPoint's supported native scripting-task activity](https://docs.evolveum.com/midpoint/reference/support-4.10/misc/bulk/scripting-tasks/) and its built-in unassign action. The same mechanism is used for native member-unassignment tasks. It explicitly deletes an expired PAM assignment, rather than relying only on a disabled record. It never unassigns the base account or ordinary application roles. Verify the unassign filter while access is active, then verify actual target removal after expiry.

The processing chain can take a scanner interval plus an expiry-task interval, in addition to execution/target delays. If a directory removal fails after the assignment has been deleted, the repeated reconciliation step still revisits the same user's desired memberships. An outage across expiry was not tested; do not infer an SLA from the nominal intervals.

After this correction, the complete 15-minute functional cycle was repeated. See the measured results for that final trial; stale-membership removal following the repair is not counted as the final automatic-expiry result.


## Windows cached-group qualification

The first post-expiry sign-out/sign-in was insufficient: the ordinary query returned Access denied, but `whoami /groups` still listed the Administrators SID as deny-only. That output alone is not a successful privilege-loss proof because UAC normally denies unelevated administrator processes too. AD independently showed an empty PAM group, no parent nesting, and only the user's ordinary application groups. Native `quser` confirmed that the earlier session had remained Active after directory revocation. The attempted query in that old elevated process was not tested because the console auto-locked; no old-token query success is claimed.

The final client validation therefore restarted only the test workstation, verified domain trust/time before sign-in, and checked the actual new user's token groups alongside the denied administrator-only query. This additional cache-clearing step is part of the recorded procedure, not a claim of exact15-minute Windows privilege termination. [Microsoft documents how cached logon information and Kerberos resources can retain stale memberships](https://learn.microsoft.com/troubleshoot/windows-client/group-policy/group-membership-changes-not-updating-over-some-vpn-connections). That article describes VPN scenarios; this lab's observed stale-group behavior is recorded independently and is not attributed to a VPN.
