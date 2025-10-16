---
layout: default
title: "Members Without Guest Invite Permission Can Add Guests to Teams"
date: 2025-10-16
tags: [security, writeup, access-control, mattermost, privilege escalation, idor]
author: "0x70da"
canonical_url: "https://0x70da.github.io/writeups/Members-Without-Guest-Invite-Permission-Can-Add-Guests-to-Teams.md"
---
## Introduction
Before I start testing any new program, I like to spend some time exploring how it works.  
I look at the main features, the structure of the application, and any role-based functions it has.  
In this case, the application was a workspace that allows admins to manage teams, permissions, and user roles.  

My goal at the beginning was simple: understand how permissions are enforced for different roles (Admin, Member, Guest).  
I explored the admin dashboard, checked which actions each role could perform, and then focused on privilege escalation scenarios especially actions that a **Member** could or could not do.


## Summary
When I start hunting a new program I first learn its main functions. In this case the app is a workspace with teams and an admin dashboard that manages roles and permissions. While testing privilege escalation, I saw that the **Member** role can have permissions like *add members* and *add guests*, and those permissions can be revoked by an admin.

![member permissions](/../../screanshots/Screenshot 2025-10-16 130610.png)

As an admin I revoked the guest-invite permission for a Member. Using the UI, attempts to invite a guest returned `403 Forbidden` as expected. Later I noticed that workspace users can exist without being assigned to any team. 

![User whithout team](/../../screanshots/Screenshot 2025-10-16 130826.png)

That led me to test the separate function that *adds an existing workspace user to a team*. using this request:

```http
POST /teams/{team_id}/members HTTP/2
Host: redacted.com
Accept: */*
Accept-Language: en
Accept-Encoding: gzip, deflate, br
X-Requested-With: XMLHttpRequest
X-Csrf-Token: 6bruxfbfdf89tpouy6s4iykggy
Content-Type: application/json
Content-Length: 79
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0
Te: trailers

{"user_id":"g4wkcwbbdpy93pniww1y76xncw","team_id":"skye3am5tpgi8eoyndhxiq7woe"}
```
by changing the `user_id` of member user to a Guest user, the request succeeded and the Guest was added to the team, even though the Member did not have the `invite_guest` permission. This shows a server-side permission check was missing for that path.

---

## Steps
1. **Learn the app**
   - Open the admin dashboard and inspect roles, teams, and permission settings to see what actions are controllable (for example: add members, invite guests).

2. **Set up**
   - Create three accounts: **Admin**, **Member**, **Guest**.

3. **Revoke the add guest permission**
   - As Admin, remove the `invite_guest` permission from the Member role.

4. **Try add guest without permission to do that**
   - While logged in as the Member, try the normal invite guest flow in the UI.
   - The UI path correctly returns `403 Forbidden` and blocks the invite.

5. **Look for other related functions**
   - Notice that some users in the workspace are not assigned to any team (they can exist in the workspace but not be members of certain teams).

6. **Test the function add existing user to team **
   - from the Member session, attempt the operation that adds an existing workspace user (member) to a team.
   - Change the id of target user (member) to id of any Guest user and send request.

7. **Observe**
   - The Guest is added to the team even though the Member role lacks `invite_guest`.

8. **Conclusion**
   - The UI and admin console enforce the permission for the usual invite flow, but a different API path (add existing user to team) lacked the same server-side check. This is a broken access control issue and lets Members effectively add Guests without permission.

---
