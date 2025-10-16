---
layout: default
title: "Members Without Guest Invite Permission Can Add Guests to Teams"
date: 2025-10-16
tags: [security, writeup, access-control, mattermost, privilege escalation, idor]
author: "0x70da"
canonical_url: "https://0x70da.github.io/writeups/Members-Without-Guest-Invite-Permission-Can-Add-Guests-to-Teams.md"
excerpt: "How I bypass invite permission while testing privilege escalation."
---
<p style="text-align:left;">
  <a href="https://0x70da.github.io" title="Back to Home" style="font-size: 24px; text-decoration: none;">
    🏠Home
  </a>
</p>
---

## Introduction

﷽

Before I start testing any new program, I like to spend some time exploring how it works.  
I look at the main features, the structure of the application, and any role-based functions it has.  
In this case, the application was a workspace that allows admins to manage teams, permissions, and user roles.  

My goal at the beginning was simple: understand how permissions are enforced for different roles (Admin, Member, Guest).  
I explored the admin dashboard, checked which actions each role could perform, and then focused on privilege escalation scenarios especially actions that a **Member** could or could not do.


## Summary
When I start hunting a new program I first learn its main functions. In this case the app is a workspace with teams and an admin dashboard that manages roles and permissions. While testing privilege escalation, I saw that the **Member** role can have permissions like *add members* and *add guests*, and those permissions can be revoked by an admin.

![member permissions](/../../screanshots/Screenshot 2025-10-16 130610.png)

this is i found:

- Admins can control what each role (Member or Guest) can do.  
- Members can have permissions like *“Add Members”* and *“Invite Guests”*.  
- These permissions can be turned on or off from the admin panel.

However, I noticed something interesting: users can exist in the workspace **without being part of any team**. 

![User whithout team](/../../screanshots/Screenshot 2025-10-16 130826.png)

The application has **two separate functions** for adding users to teams:

1. **Invite function** — used to invite a person who is not yet in the workspace (new member or guest). 

2. **Add existing user to team function** — used to add users who already exist in the workspace but are not currently assigned to the team.

When the Member role hasn't permission to add *guests*, the first (invite) flow is blocked correctly (returns `403 Forbidden`) when i tried to invite guests, This path performs a permission check and will block a Member who lacks the `invite_guest` permission.

So i decided to test the “add existing user to team function”.


This is request for *adding an existing workspace user to a team*:

```http
POST /teams/{team_id}/members HTTP/2
Host: redacted.com
Accept: */*
Accept-Language: en
Accept-Encoding: gzip, deflate, br
X-Requested-With: XMLHttpRequest
Content-Type: application/json
Content-Length: 79
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0
Te: trailers

{"user_id":"{user_id}","team_id":"{team_id}"}
```
but this request work if the member has permission to add members, so the one that came to my mind is changing the `user_id` of member user to id of Guest user, now i send the request and i observed that the request succeeded and the Guest was added to the team, even though the Member did not have the `invite_guest` permission. This shows a server side permission check was missing for that path.

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

6. **Test the function add existing user to team**
   - from the Member session, attempt the operation that adds an existing workspace user (member) to a team.
   - Change the id of target user (member) to id of any Guest user and send request.

7. **Observe**
   - The Guest is added to the team even though the Member role lacks `invite_guest`.

8. **Conclusion**
   - The UI and admin console enforce the permission for the usual invite flow, but a different API path (add existing user to team) lacked the same server-side check. This is a broken access control issue and lets Members effectively add Guests without permission.

### Here’s the short version:
- I revoked the `invite_guest` permission from a Member role.
- I confirmed the normal “invite guest” feature was blocked `403 Forbidden`.
- Later, I Observed another function: adding existing users (members) to a team.
- And when i change the `user_id` to guest user, the request is successed.
- The Member could add a Guest user successfully even though they shouldn’t have permission.

This means the permission check was only enforced in the main invite flow, and the API level check if the member can add another members teams,  but it isn't check the invite permission even if when the target user is guest. 

## Impact:
- A Member can add Guest users to teams even when the Member role does not have the invite_guest permission.
- This can lead to unauthorized access to teams and potentially to sensitive information or resources that Guests gain by being added.

## Vulnerability classification
- VRT: **Broken Access Control (BAC) → Privilege Escalation**  
- Priority: **P3**

### Thank You
Thank you for reading my write-up. I hope this helps others understand how small differences in permission enforcement can lead to privilege escalation vulnerabilities.
---
