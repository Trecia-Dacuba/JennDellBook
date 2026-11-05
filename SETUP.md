# JendellBook: setup

## A. Try it right now (no setup)
Open `jendellbook.html` in Chrome or Edge. It runs in **demo mode**: everything you do is saved in
that browser (reload and your bookings are still there; open it in two tabs and they stay in sync).
Demo logins are shown on the login screen. "Reset demo data" in the grey banner starts over.

## B. Switch to the real Firebase database (about 10 minutes)
1. console.firebase.google.com > **Add project**.
2. **Build > Authentication > Get started** > enable **Email/Password**.
3. **Build > Firestore Database > Create database** (production mode). Open the **Rules** tab and paste `firestore.rules`.
4. **Build > Storage > Get started**. Open the **Rules** tab and paste `storage.rules`.
   (Google may ask you to switch to the Blaze pay-as-you-go plan for Storage; it has a free allowance.
   Screenshots need Storage, so this step is required for payment proofs.)
5. **Project settings > Your apps > Web (</>)**: register an app and copy the `firebaseConfig` object.
6. In `jendellbook.html`, replace `const FIREBASE_CONFIG = null;` with `const FIREBASE_CONFIG = { ...your config... };`
7. **Create the first owner login** (one time, in the console):
   - Authentication > Users > **Add user** (email + password). Copy its **User UID**.
   - Firestore > **Start collection** `users` > Document ID = that UID > fields:
     `email` (string), `name` (string, e.g. Owner), `role` (string) = `admin`.
8. Host the page so it is not opened from a `file://` path: Firebase Hosting, GitHub Pages, or `npx serve` on localhost.
   Then Authentication > Settings > **Authorized domains**: add your domain.
9. Log in on the **Staff** tab as the owner. In Administration click **Load sample services and
   therapists**, then create front-desk and therapist accounts from the same page.
   Clients register themselves from the Client tab.

## The hold timer
The client has the "hold" time (default 30 minutes) to send the GCash proof. **When the proof is sent the
timer stops** and the slot stays reserved until staff verify or reject it. If staff reject, the client gets a
fresh hold window to resubmit. The owner can optionally set "Staff must verify within (hours)" in
Administration: if staff have not decided by then, the slot is released. 0 means no limit.

## How double-booking is prevented
Each therapist-hour is a document in `slots` with an id like `therapistId_2026-11-02_1400`.
A booking creates every slot it needs inside **one Firestore transaction**; if any of them is already
taken (and not expired) the whole booking is refused. A 90-minute service locks two hours.
The rules in `firestore.rules` stop a client from taking a slot that is still live.

## Things to know
- Your design document uses Cloud Functions for the transaction. Cloud Functions need the pay-as-you-go
  plan, so this version runs the same transaction from the page, protected by the security rules.
  The transaction code (`book`, `decide`, `release`) can be moved into callable functions later.
- Held slots that run out of time are released by the next staff or client browser that sees them
  (no server needed). A scheduled Cloud Function would be the always-on version.
- SMS messages are recorded and shown in the app, not actually sent. Sending needs a server-side
  call to Semaphore (a Cloud Function), because the API key must not be in the page.
- Removing an account deletes its profile only; delete the login itself in Authentication.
- Review `firestore.rules` with your adviser and test it in the Rules Playground before real customers use it.

## Choosing an SMS service (prices checked Nov 2026; confirm before you rely on them)
| Option | Cost | Good for | Watch out for |
|---|---|---|---|
| **textbee.dev** (your own Android phone + SIM) | Free plan: 300 SMS/month, 50 per day, 1 device. You pay only your own load or unli-text. | Demo, defense, and a quiet spa | Phone must stay on, online and charged. Texts come from that SIM's number. Carrier fair-use limits. |
| **Semaphore** (Philippine SMS API) | About ₱0.56 per text (ex. VAT), prepaid credits, no monthly fee. First sender name is free but needs approval; until then texts show "SEMAPHORE". | Real daily use | Needs a small server-side function to hold the API key. |
| **PhilSMS** | Advertised near ₱0.35 per text on one comparison site; verify on their own site. | Cheapest local API | Less documentation than Semaphore. |
| Twilio / Vonage trial | Free credit, but trial accounts send only to verified numbers. | Not suitable for real clients | |

Rough cost: 20 bookings a day with 2 texts each is about 1,200 texts a month, around ₱670 on Semaphore.
Sending only the confirmation (about 600 texts) is around ₱340. Each booking already brings in a ₱100 deposit.

Whichever you pick, the API key must never be written into `jendellbook.html`. Call the provider from a
Cloud Function or a small server. Because only the Notification module talks to the SMS provider
(your architecture diagram), switching providers later changes one module and nothing else.
Also tell clients at sign-up that you will text them (Data Privacy Act consent).
