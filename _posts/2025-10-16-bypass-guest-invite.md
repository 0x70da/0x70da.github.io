---
layout: default
title: "Members Without Guest Invite Permission Can Add Guests to Teams"
date: 2025-10-16
tags: [security, writeup, access-control, mattermost, privilege escalation, idor]
author: "0x70da"
canonical_url: "https://0x70da.github.io/writeups/Members-Without-Guest-Invite-Permission-Can-Add-Guests-to-Teams.md"
---

## Summary
When I start hunting a new program I first learn its main functions. In this case the app is a workspace with teams and an admin dashboard that manages roles and permissions. While testing privilege escalation, I saw that the **Member** role can have permissions like *add members* and *add guests*, and those permissions can be revoked by an admin.

As an admin I revoked the guest-invite permission for a Member. Using the UI, attempts to invite a guest returned `403 Forbidden` as expected. Later I noticed that workspace users can exist without being assigned to any team. That led me to test the separate function that *adds an existing workspace user to a team*. In my lab, I tried that add-to-team flow from a Member account: by changing the target user to a Guest account, the request succeeded and the Guest was added to the team — even though the Member did not have the `invite_guest` permission. This shows a server-side permission check was missing for that path.

> All testing described here was done only in an isolated test environment (sandbox/staging). Do not run these steps against production systems without written permission.

---

## Steps I followed (simple, safe)
1. **Learn the app**
   - Open the admin dashboard and inspect roles, teams, and permission settings to see what actions are controllable (for example: add members, invite guests).

2. **Set up a lab**
   - Use a sandbox or staging instance.
   - Create three accounts: **Admin**, **Member**, **Guest**.

3. **Revoke the guest-invite permission**
   - As Admin, remove the `invite_guest` permission from the Member role.
   - Verify the admin console shows the permission revoked.

4. **Check the UI behavior**
   - While logged in as the Member, try the normal invite flow in the UI.
   - The UI path correctly returns `403 Forbidden` and blocks the invite.

5. **Look for other related functions**
   - Notice that some users in the workspace are not assigned to any team (they can exist in the workspace but not be members of certain teams).
   - Map the client-side network traffic to find the API endpoints the UI uses for team membership (use DevTools to *observe only*).

6. **Test the add-existing-user-to-team flow (lab only)**
   - In the sandbox, from the Member session, attempt the operation that adds an existing workspace user to a team.
   - Use safe, controlled requests in your test environment (do not expose real cookies/keys).
   - Change the target user to a Guest account and perform the add-to-team action.

7. **Observe the result**
   - The Guest is added to the team even though the Member role lacks `invite_guest`.
   - Capture sanitized evidence: admin-perms screenshot, team membership screenshot, server logs (redacted).

8. **Conclusion**
   - The UI and admin console enforce the permission for the usual invite flow, but a different API path (add existing user to team) lacked the same server-side check. This is a broken access control issue and lets Members effectively add Guests without permission.

---

## Short mitigation notes for developers
- Enforce permission checks on the **server** for every endpoint that adds or invites users. Do not rely on the UI to enforce permissions.
- Centralize authorization: route all membership/invite operations through one authorization helper.
- Add tests that cover revoked-permission scenarios and expect `403 Forbidden` where appropriate.
- Log permission checks and failures clearly.

---
