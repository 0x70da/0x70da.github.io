---
layout: post
title: "Members Without Guest Invite Permission Can Add Guests to Teams"
date: 2025-10-16
tags: [security, writeup, access-control, mattermost, privilege escalation, idor]
author: "0x70da"
canonical_url: "https://0x70da.github.io/writeups/Members-Without-Guest-Invite-Permission-Can-Add-Guests-to-Teams.md"
---

## Summary
A privilege escalation / broken access control issue allowed regular workspace members — even after the `invite_guest` permission was revoked — to add guest users to specific teams. This bypass of role-based restrictions violates the principle of least privilege and permitted unauthorized access to resources scoped to teams.


---

## Vulnerability classification
- VRT: **Broken Access Control (BAC) → Privilege Escalation**  
- Priority: **P3**

---

## Impact
- Members who should not have the ability to invite guests were able to add guest accounts to teams.  
- This leads to unauthorized access to team channels and resources that were intended to be restricted.  
- Business impact includes data exposure risk, unauthorized collaboration, and possible lateral movement depending on guest privileges.

---

## Root cause (high level)
The server-side enforcement of the `invite_guest` permission was incomplete for the team-membership API. Although workspace-level permission settings in the admin console could remove the `invite_guest` capability for member-role users, the API endpoint that adds members to a team did not verify the caller’s effective permission correctly — allowing members to perform an operation that should have been prohibited.

---

## Safe reproduction (for testers / developers)
> **Important:** Do **not** execute any of these steps against systems you do not own or have explicit written permission to test. Run the steps only on a local instance, a staging environment, or an authorized target.

1. **Prepare a test environment**
   - Create three accounts:
     - *Admin* — with full admin privileges.
     - *Member* — the role you will restrict.
     - *Guest* — an account that should only become a member of a team if invited by an authorized role.

2. **Configure permissions**
   - As *Admin*, go to the admin console → User Management → Permissions → System Scheme (or equivalent) and **revoke** the `invite_guest` permission from the *Member* role.

3. **Invite and join**
   - Have the *Admin* invite the *Member* (or invite the *Member* through a normal workflow) and ensure the *Member* accepts and is a regular member (not admin).

4. **Gather identifiers (for testing only)**
   - From the test instance, list users to obtain the internal user IDs and list teams to obtain team IDs. (Use admin-level API or the test UI; do not ex
