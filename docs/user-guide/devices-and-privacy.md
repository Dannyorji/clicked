# Devices, safety numbers, and privacy

A plain-language guide to what Clicked's encryption actually protects, how to manage the
devices signed in to your account, how to confirm you're really talking to who you think you
are, and what to do when something looks off.

This is the non-technical companion to [`docs/threat-model.md`](../threat-model.md), which is
written for engineers and auditors. Everything in the first section below is meant to agree
with that document — if the two ever seem to say different things, the threat model is the
more precise, more current source, and this page should be corrected to match it.

---

## What Clicked can and cannot see

Clicked encrypts your messages end-to-end. In practice that means two very different things
are true at the same time, and it matters which one applies to any given piece of information.

### Clicked cannot read your messages

When you send a text message, a photo, a video, or a file, it is scrambled (encrypted) on
your own device before it ever leaves it. It only becomes readable again on the recipient's
device. Clicked's servers move that scrambled data around, but they never hold the key needed
to unscramble it — that key only ever exists on your devices and the devices of the people
you're talking to.

This covers:

- The actual words, images, and files in your messages.
- The keys that would be needed to decrypt them. Clicked's servers have no field in any
  database table that could even hold one of your private keys — it isn't a policy decision
  that could be changed by a setting, it's that the capability was never built.
- The internal "session" state your app keeps to encrypt and decrypt — that lives only on
  your device.

### Clicked does see some things — this is normal for any messaging app

For Clicked to deliver a message to the right person, at the right time, on the right device,
it has to know some things in plain, unencrypted form. This is true of every messaging service
that routes messages between people, not a Clicked-specific gap:

| What Clicked sees | Why it has to |
| --- | --- |
| Who you're messaging, and who's in a group with you | It has to know where to deliver a message |
| When you send a message, and roughly how big it is | Delivery and storage both need this; a large file is visibly large even though its contents aren't readable |
| Whether a message is text, an image, a file, etc. | Used to show the right icon/preview before anything is decrypted |
| Whether you're online, and when you were last active | This is how "online" status and delivery receipts work — you can turn your own visibility off in your privacy settings, but Clicked's servers still see it to run the feature |
| The public keys your devices use to start encrypted conversations | These are *public* by design — sharing them is how two devices agree on a shared secret in the first place. Your *private* keys never leave your device |
| Your device names and device type (phone, web, etc.) | Shown to you on the [devices page](#linking-and-revoking-devices) so you can recognize your own devices |
| Your wallet address | Used to sign you in and to identify you for on-chain treasury and payment features |

None of this lets Clicked read what you actually said. But taken together, this kind of
information — called **metadata** — can still reveal a lot: who you talk to, how often, and
roughly when, even without anyone reading a single message. If someone fully compromised
Clicked's servers, this is the kind of information they would get access to; they still would
not get your past message contents without separately compromising one of your devices too.

### The one deliberate exception: the AI assistant

Clicked's AI assistant is a chat feature you can ask questions. **Anything you send to the AI
assistant is not end-to-end encrypted.** The question you type is sent as plain, readable text
to Clicked's backend and from there to an outside AI provider (OpenAI) to generate a reply —
the same way the question would be visible if you emailed it to a support team. The assistant's
reply is also stored in a way Clicked's own servers can read (#363).

Use the AI assistant for general questions about how Clicked works — don't use it to ask
questions that contain anything you wouldn't want outside the encrypted part of the app, the
same way you wouldn't paste a sensitive message into a generic search engine.

### What this doesn't protect you from

Encryption protects the content of your conversations. It does not:

- Hide *that* you're using Clicked, or *who* you talk to, from Clicked itself (see the metadata
  table above).
- Protect you if someone else has physical access to your already-unlocked device — anyone
  looking at your unlocked phone can read messages the same way you can.
- Protect messages after they're decrypted and displayed — a screenshot, or someone reading
  over your shoulder, bypasses encryption entirely, because at that point the message is just
  text on a screen.

---

## Linking and revoking devices

You can be signed in to Clicked on more than one device at once (phone, web browser, etc.).
Each one is tracked separately so you can see exactly what's signed in and shut off any device
you don't recognize or no longer use.

### Linking a new device

There's no separate "pairing" step — open Clicked on the new device and connect the same
wallet you use everywhere else. The device registers itself automatically and appears in your
device list (**Settings → Linked Devices**) the moment it signs in.

### Seeing what's linked

Your device list shows, for every device signed in to your account:

- A name and device type (web, iOS, Android)
- When it was last active
- Whether it's the device you're currently using

### Revoking a device

If you lose a device, retire an old one, or see one you don't recognize, revoke it from
**Settings → Linked Devices**. Revoking:

- Signs that device out immediately.
- Rotates the keys that device was using, so it can no longer decrypt any *new* messages sent
  to you after the revocation.

You can also **log out everywhere else** in one action, which revokes every device except the
one you're currently using — useful if you're not sure which specific device to revoke, or
you believe your account was accessed from somewhere you don't recognize.

**A key-change notice afterward is expected, not a sign of a problem.** Revoking a device
changes the keys associated with your account. The next time someone messages you, their app
may show them a notice that your keys changed — that's the system working correctly, confirming
the old, revoked device genuinely can no longer read new messages. See
[Verifying a contact's safety number](#verifying-a-contacts-safety-number) below for what to do
if *you* see that notice about someone else.

### Why a newly linked device can't read older group history

This surprises people the first time they see it: link a new device, open a group conversation
you were already part of, and the messages from before you linked that device aren't there —
they show as unavailable on that device specifically, not as an error.

This is a direct consequence of how group encryption works, not a bug or a missing sync step.
Group conversations derive a fresh encryption key every time the group's membership changes
(someone joins, someone leaves, a device is added). A device only receives the key material
starting from the point it joined — there is no copy of the old keys sitting on Clicked's
servers that could be handed to it, because Clicked's servers never had them either. The same
property is what makes removing a device from a group meaningful: if a newly added device
could reach backward and decrypt everything before it joined, so could a device that was
removed and then sneakily re-added, which would defeat the point of removing it.

In short: a new device sees everything from the moment it joined onward, and nothing before —
by design, as the price of the removal/rotation step actually being secure. Your other
devices, and the other group members, still have that older history; it isn't lost, just not
visible on that particular new device.

### After losing a device

If a phone or laptop with Clicked signed in is lost or stolen:

1. **Revoke it immediately** from another device, via **Settings → Linked Devices**, or use
   **log out everywhere else** if you're not sure the lost device is the only one you no longer
   trust.
2. If you can't get to another signed-in device, sign in again on a device you do trust — this
   issues you a new device identity, and you can revoke the lost one from there once you can
   see it in the list.
3. Expect the people you talk to most to briefly see a "security code changed" notice the next
   time you message them from your remaining devices — this is them seeing evidence that your
   device set changed, which is exactly what should happen. Nothing further is required on
   their end beyond the ordinary verification described below, and it does not mean they were
   compromised.
4. Revoking does not retroactively protect messages the lost device already decrypted and
   displayed before you revoked it — if someone has the unlocked device itself, revocation
   stops *new* messages, not what was already on the screen. Treat a lost, unlocked device the
   same way you'd treat losing an unlocked phone for any other sensitive account.

---

## Verifying a contact's safety number

A **safety number** is a long string of digits that's computed from the encryption keys of a
specific person's devices. Its only job is to answer one question: *am I really encrypting to
the devices this person controls, or has something changed underneath me?*

### Why this exists

Every messaging app that does key exchange through a central server has the same theoretical
weak point: the server is the one telling you which public keys belong to your contact. If a
server were ever compromised in a very specific way, it could in principle try to hand you the
wrong keys without you noticing. Comparing safety numbers closes that gap, because it gives you
a way to confirm, independently of the server, that you and your contact are both looking at
the same keys.

For most people, most of the time, this isn't something you need to actively think about —
Clicked checks it automatically on your behalf (see the next section). Explicit verification
matters most for conversations where confirming identity really counts.

### How to verify

1. Open the conversation with the contact you want to verify.
2. Open **Safety numbers** for that conversation.
3. Compare the number shown with the same number on your contact's device — read it aloud on a
   call, compare it in person, or send it over a different channel than Clicked itself (the
   point is to use a path other than the one you're trying to verify).
4. If it matches, tap **Mark verified**.

### "Safety number changed" / "security code changed"

Clicked automatically remembers the keys it has seen for each of your contacts. If those keys
ever change — because your contact linked a new device, revoked an old one, or (far less
likely) something is actually wrong — you'll see a **"Safety number changed"** notice the next
time you'd otherwise send them a message, and sending is paused until you deal with it.

What to do:

1. **Don't panic — check the ordinary explanation first.** The overwhelmingly common cause is
   that your contact linked or revoked a device, which is completely normal account maintenance
   on their end. Ask them, through any channel, whether they recently added or removed a
   device.
2. **Re-verify the safety number** using the steps above, against their *current* devices.
3. Once it matches, tap **Mark verified** (or re-verify) to clear the warning and resume
   sending. Until you do, Clicked deliberately keeps refusing to send — it fails closed rather
   than silently encrypting to keys it hasn't confirmed, because a changed key is exactly the
   situation this check exists to catch.

If your contact says they *didn't* link or revoke anything and the number still doesn't match
after checking again, treat that as a real signal something is wrong and stop sending sensitive
information in that conversation until it's resolved.

---

## See also

- [`docs/threat-model.md`](../threat-model.md) — the technical version of the "what Clicked can
  and cannot see" section above, written for engineers.
- [`docs/data-retention.md`](../data-retention.md) — how long your messages, devices, and other
  data are kept, and what deleting something does and doesn't remove.
