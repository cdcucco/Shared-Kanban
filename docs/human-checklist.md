# SharedKanban human checklist

This is the only SharedKanban document you must personally act on. It lists every one-time action that truly needs a human: a password, a fingerprint, a payment card, a plug, or a decision that is yours. Everything else is done by Claude in the cloud (writing code and documents) or by a Cowork session on the Mac mini (building, testing, installing and operating), as docs/operator-playbook.md describes (D20). Secrets are never typed into chat: when a step produces a password or a key file, this list says where it goes on the Mac mini, and a Cowork session names the file and waits rather than asking for the value. Each action is numbered taps with the menu names you will see; no knowledge of Apple's developer tools is assumed. "Needed before phase N" means that phase's review cannot pass without it.

## Summary

| # | Action | Needed before phase | Estimated minutes | Can a Cowork session sit beside you and do most of it? |
|---|---|---|---|---|
| 1 | Sign in with your Apple Account on the Mac mini and in Xcode; accept the Xcode license | 1 | 15 | Yes |
| 2 | Enroll in the Apple Developer Program; accept the App Store Connect agreements (start now; takes days) | 3 (start during 1) | 20, plus waiting days | No |
| 3 | Approve the App ID capabilities; create and download the three keys; put each .p8 where Cowork says | 3 (keys also used in 8 and 11) | 30 | Yes |
| 4 | iPhone and iPad: Developer Mode, trust the Mac by cable, TestFlight invitation, allow notifications and Sign in with Apple | 3 (TestFlight in 11) | 20 per device | No |
| 5 | Tailscale account; sign in on the Mac mini and both devices; enable HTTPS certificates; disable key expiry for the Mac mini | 10 | 25 | Yes |
| 6 | Type the macOS admin password when Docker Desktop, Tailscale, auto-login, FileVault and energy settings ask | 1 and 10 | 10, in several moments | Yes |
| 7 | Plug in a UPS and an external backup SSD; confirm iCloud Drive has enough space | 10 | 20, plus shopping | No |
| 8 | Save the restic password and the secrets-image passphrase in your password manager | 10 | 10 | Yes |
| 9 | Optional: let the cloud session compile Swift (edit the Claude cloud environment) | 1 or any time | 5 | No |
| 10 | Optional, later: a Backblaze B2 account; pasting the Claude connection token by hand | Only if asked | 10 | No |
| 11 | Reply "approved" or list changes at each phase review | Every phase | 15 per phase | No |

## 1. Sign in with your Apple Account on the Mac mini and in Xcode

Why: Xcode can only build and sign the app for real devices when it is signed in with the Apple Account that owns the developer membership (D20).

When: before the Phase 1 review.

Before you start: the Mac mini switched on, your Apple Account (Apple ID) email and password, and your iPhone nearby for the two-factor code. Use the same account for every Apple step in this list.

Steps:

1. Log in to the macOS user account Cowork names.
2. Open the Apple menu, then System Settings.
3. At the top of the sidebar click Sign in.
4. Type your Apple Account email and password, click Next.
5. On your iPhone tap Allow, then type the six-digit code on the Mac mini.
6. If the App Store asks you to sign in while Cowork installs Xcode, use the same account.
7. Open Xcode. If a window titled Xcode and Apple SDKs Agreement appears, click Agree and type the admin password.
8. In Xcode open the Xcode menu, Settings, Accounts.
9. Click + at the bottom left, choose Apple Account, click Continue, sign in, approve the code again if asked.
10. Tell Cowork "signed in".

What to hand back: nothing. Password and code go only into Apple's windows; the session lives in the Mac mini login keychain, which Cowork keeps unlocked for signing (D20).

How you know it worked: System Settings shows your name in the sidebar, Xcode's Accounts list shows your email with a team under it, and Cowork reports a build that asked for nothing. Xcode may occasionally ask again; Cowork will say when (D20).

## 2. Enroll in the Apple Developer Program

Why: Sign in with Apple, push notifications and TestFlight only work for a paid membership, and Apple can take days to approve it (D20).

When: start during Phase 1; needed before the Phase 3 review.

Before you start: the Apple Account from action 1, a payment card for the yearly fee, and possibly a government ID.

Steps:

1. On your iPhone or the Mac mini open Safari and go to developer.apple.com.
2. Tap Account and sign in.
3. Tap Join the Apple Developer Program, then Enroll.
4. Choose Individual and confirm your name and address.
5. If Apple asks you to confirm your identity, follow its prompts (sometimes inside the Apple Developer app on the iPhone).
6. Read the license agreement, tick the box, tap Continue.
7. Pay the yearly fee with your card.
8. Wait for the email saying your membership is active (hours or days).
9. Go to appstoreconnect.apple.com and sign in.
10. If a yellow banner says an agreement needs your attention, open it, tick the box and tap Agree. Repeat until no banner remains; only the free-apps agreement is needed.
11. Tell Cowork "enrolled".

What to hand back: nothing secret. Your Team ID (a ten-character code on the Membership page) is not a secret and is fine to read out.

How you know it worked: the Membership page shows an expiry date one year away, and App Store Connect shows no pending banners.

## 3. Approve the App ID capabilities and create the three keys

Why: the app needs three switches on its App ID and three keys only a human may create: one lets the server verify Sign in with Apple (D09), one lets it send push notifications (D07), and one lets Cowork upload to TestFlight unattended (D20).

When: before the Phase 3 review. The APNs key is used from Phase 8 and the App Store Connect key from Phase 11, but one sitting is simplest. Cowork asks when it is ready.

Before you start: action 2 complete, Cowork running on the Mac mini with the browser open, you at the Mac mini so downloads land there. Each key downloads exactly once; if a file is lost, revoke it and create a new one.

Steps:

1. When Xcode (driven by Cowork) asks to register the App ID or enable a capability, click Enable or Register. If Cowork instead opens developer.apple.com, Identifiers: open the SharedKanban App ID, tick Sign in with Apple, Push Notifications and Associated Domains, click Save, Confirm.
2. In the sidebar click Keys, then +.
3. Name it SharedKanban Sign in with Apple, tick Sign in with Apple, click Configure, choose the SharedKanban App ID as the primary App ID, click Save, Continue, Register.
4. Click Download. Leave the file in Downloads; do not open it.
5. Click Keys and + again. Name it SharedKanban APNs, tick Apple Push Notifications service (APNs), click Continue, Register, Download.
6. Go to appstoreconnect.apple.com, click Users and Access, the Integrations tab (older layouts say Keys), App Store Connect API, Team Keys.
7. Click +, name it SharedKanban Cowork, choose the access level App Manager, click Generate, then Download.
8. Tell Cowork "three files downloaded". Cowork moves them from Downloads into the secrets folder outside the repository, named `sign-in-with-apple.p8`, `apns.p8` and `app-store-connect-api.p8`: `~/SharedKanban/secrets/` until Phase 10, `/Users/kanban/SharedKanban/secrets/` after it (D11, D21). If you downloaded on another device, AirDrop the files to the Mac mini first.
9. Leave the Keys pages open: Cowork reads the Key ID, Issuer ID and Team ID from them into the `.env` file in the same folder.

What to hand back: never a .p8 file's contents, and nothing in chat. The files live only in the secrets folder and the encrypted secrets image on the backup SSD (D21).

How you know it worked: the Keys page lists two keys, the App Store Connect API page lists the third, Downloads holds no `AuthKey` file, and Cowork reports that `.env` names all three.

## 4. Prepare the iPhone and the iPad

Why: a Mac can only install builds on a device with Developer Mode on that has trusted it once; TestFlight, notifications and Sign in with Apple each ask the person holding the device (D20).

When: Developer Mode and trust before the Phase 3 review; notifications in Phase 8; TestFlight in Phase 11. Repeat on each device.

Before you start: the device unlocked, on iOS or iPadOS 18 or later (verify at Phase 1, D20), its passcode, a USB-C cable that fits both ends, and Cowork saying it is ready.

Steps:

1. Plug the device into the Mac mini.
2. On the device tap Trust on "Trust This Computer?" and type the passcode.
3. Open Settings, Privacy & Security, scroll to the bottom, tap Developer Mode (the row appears only after step 2).
4. Turn it on, tap Restart, and after the restart tap Turn On and type the passcode.
5. Tell Cowork "device ready" and keep the cable connected while it installs.
6. When the app shows Sign in with Apple, tap it, then Continue with Face ID or Touch ID. Share My Email or Hide My Email are both fine (D04).
7. When the app explains why it wants notifications and iOS asks "Allow notifications?", tap Allow (D07).
8. When the TestFlight invitation arrives (Phase 11), install TestFlight from the App Store, open the link, tap Accept, then Install.

What to hand back: nothing. The passcode and Face ID stay on the device.

How you know it worked: Privacy & Security shows Developer Mode on; SharedKanban opens to the board after sign-in; Settings, Notifications lists it as allowed; in Phase 11 TestFlight lists it as installed.

## 5. Set up Tailscale

Why: release-1 production runs over a private Tailscale network, so nothing is on the public internet and no domain is needed; the two admin-console switches give the Mac mini a trusted HTTPS certificate and keep it connected without re-authentication (D10).

When: before the Phase 10 review. Earlier phases use the home network only.

Before you start: an email address, the Mac mini with Cowork running, both devices in hand.

Steps:

1. Go to tailscale.com, click Get started, and sign up with the login provider you prefer (your Apple Account or Google). Use the same provider on every device.
2. On the Mac mini, after Cowork installs Tailscale, click Log in in its menu-bar icon, then Connect in the browser. If macOS asks to allow an extension or VPN configuration, click Allow and type the admin password.
3. On the iPhone, install Tailscale from the App Store, open it, tap Log in, sign in, and tap Allow when iOS asks to add a VPN configuration.
4. In the Tailscale app's Settings turn VPN On Demand on (D10).
5. Repeat steps 3 and 4 on the iPad.
6. On the Mac mini, with Cowork guiding, open login.tailscale.com, click the DNS tab, find HTTPS Certificates, click Enable HTTPS, confirm.
7. Click the Machines tab, find the Mac mini row, open its three-dot menu, choose Disable key expiry, confirm.
8. Tell Cowork "Tailscale done". Do not enable Funnel: it puts the server on the public internet and is turned on only if you ever want access without Tailscale, for example from claude.ai (D10, D15); then Cowork opens the Funnel prompt and you click Enable.

What to hand back: nothing. Cowork reads the tailnet hostname on the Mac mini; it never enters the repository (D10).

How you know it worked: the Machines page lists all three devices, the Mac mini row says "Expiry disabled", the DNS page shows HTTPS Certificates enabled, and the Phase 10 results note shows both devices reaching the server over https.

## 6. Type the macOS admin password when asked

Why: installing Docker Desktop and Tailscale and creating the auto-login service user change system settings that macOS accepts only from a human typing the admin password (D11, D21).

When: Phase 1 (Docker Desktop's first launch, the Xcode license) and Phase 10 (service user, auto-login, FileVault, energy settings).

Before you start: the Mac mini admin password, and your FileVault decision (step 7).

Steps:

1. Whenever a macOS window says "wants to make changes" or asks for your password, type the admin password and click OK. Cowork warns you before each one.
2. Docker Desktop, first launch: click Accept on its terms, Allow for privileged access, type the password.
3. Tailscale: if System Settings, General, Login Items & Extensions shows a blocked extension, click Allow.
4. Service user: when Cowork creates the user kanban in System Settings, Users & Groups, type the admin password, choose a password for kanban, and store it in your password manager.
5. Auto-login: in Users & Groups, set Automatically log in as to kanban and type its password (D21).
6. Energy: in System Settings, Energy, turn on Start up automatically after a power failure and Prevent automatic sleeping when the display is off (D21).
7. FileVault: in Privacy & Security, FileVault, leave it Off unless you want the disk encrypted. Off lets the Mac mini restart by itself after a power cut; On means every full restart waits for a typed password (D21).
8. In your Phase 10 reply write "FileVault off" or "FileVault on".

What to hand back: only the FileVault decision, in the phase reply.

How you know it worked: Docker Desktop and Tailscale show as running, the Mac mini logs in as kanban by itself after a restart, and the Phase 10 results note records your choice.

## 7. Plug in a UPS and an external SSD, and check iCloud Drive

Why: the Mac mini is the only server, so power protection and a local encrypted backup are part of the product; iCloud Drive holds the off-site copy (D21).

When: before the Phase 10 review.

Before you start: a UPS with a USB port, an external SSD (Cowork tells you the minimum size before you buy), its cable, and iCloud signed in on the Mac mini.

Steps:

1. Plug the Mac mini into a battery-backed outlet on the UPS and the UPS into the wall.
2. Connect the UPS's USB cable to the Mac mini; Cowork sets the UPS options.
3. Plug in the SSD.
4. When Cowork asks to erase it (Disk Utility shows an Erase sheet named KanbanBackup), click Erase. Everything on the drive is deleted.
5. Open System Settings, click your name, iCloud, Manage, and note the free space. Cowork tells you how much the backup copy needs; if you have less, click Change Storage Plan and choose a larger one.
6. In the same screen open iCloud Drive and turn Optimize Mac Storage off (D21).
7. Tell Cowork "UPS, SSD and iCloud done".

What to hand back: nothing.

How you know it worked: Finder shows KanbanBackup, Energy shows the UPS, and after Phase 10 the app's Server status screen shows a recent backup and a passed restore drill (D21).

## 8. Save two passwords in your password manager

Why: backups are encrypted with the restic password, and the secrets folder is backed up as an encrypted image with its own passphrase; losing either loses what it protects (D21).

When: before the Phase 10 review, right after Cowork generates them.

Before you start: your password manager open on any device; Cowork running on the Mac mini.

Steps:

1. Cowork generates the restic password into `restic-password` in the secrets folder and opens it on the Mac mini screen (D21).
2. Create an entry named SharedKanban restic and paste the password into it, copied from the Mac mini screen, never from chat.
3. Cowork runs `deploy/backup/export-secrets.sh` and shows the passphrase it created for the secrets image.
4. Create an entry named SharedKanban secrets image and paste that passphrase.
5. Close the files on the Mac mini and tell Cowork "saved".

What to hand back: the word "saved". Neither password is ever typed into chat.

How you know it worked: both entries open on another device, and the Phase 10 restore drill passes (D21).

## 9. Optional: let the cloud session compile Swift

Why: the cloud Linux session writes the server code but can only build and test it if its environment may download the Swift toolchain; otherwise GitHub Actions and the Mac mini compile, one extra round trip per phase (D20, docs/operator-playbook.md section 4).

When: any time from Phase 1; skipping it blocks nothing.

Before you start: a Claude Code cloud session open in your browser.

Steps:

1. In the session title bar open the cloud environment menu and choose Edit.
2. Under Network access add `download.swift.org`, or choose a broader access level if a single-domain allowlist is not offered.
3. Under Setup script paste the Swift install command that docs/operator-playbook.md section 4 names for the approach chosen at Phase 1.
4. Click Save.

What to hand back: nothing; there is no secret here.

How you know it worked: the Phase 1 cloud session reports that `swift test` ran in `shared/` and `server/`.

## 10. Optional, later: a Backblaze B2 account and the Claude connection token

Why: B2 is the off-site backup fallback only if a week of monitored runs shows iCloud Drive unreliable (D21). Cowork normally pairs the Claude client itself with a token it creates on the Mac mini, so no human step exists (D15).

When: only if a Phase 9 or Phase 10 results note asks for it.

Before you start: the results note that asks; for B2, a payment card.

Steps:

1. B2: create a Backblaze account, add a card, tell Cowork "B2 ready". Cowork creates the bucket and key with you and files the key in the secrets folder.
2. Connection token: if Cowork cannot paste the token into the Claude client, open the client's settings on the Mac mini, paste the token Cowork shows on screen into the field it names, click Save. Never copy it into chat (D15).

What to hand back: "B2 ready" or "token pasted".

How you know it worked: the next results note shows the B2 copy succeeding, or the Claude client lists the SharedKanban tools.

## 11. Reply at each phase review

Why: no phase starts until you have read its review and replied (D20).

When: at the end of every phase, 0 through 11.

Before you start: docs/reviews/phase-N-review.md for the phase just finished.

Steps:

1. Read the summary at the top and the list of decisions and known risks.
2. If you are happy, reply with the single word "approved".
3. If not, reply with a numbered list of the changes you want, in plain words.
4. If the review lists an action from this list as still open, do it first.

What to hand back: the reply itself. It never needs a password, a key or a hostname.

How you know it worked: the cloud session merges the phase and announces the next one.

## Right now (Phase 0)

Nothing except reading docs/reviews/phase-0-review.md and replying "approved" or listing changes.
