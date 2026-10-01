# silnet spec (draft 0)

> **Note:** every size in this spec is an estimate. The benchmark (`bench/sizes.py`) will measure the real numbers. If they come out significantly better or worse, this spec gets updated to match.
>
> **How to use this file:** sections 1 to 7 are what Phases 1 and 2 need. Sections 8 to 14 are first drafts. Revisit them when you reach those phases, and change anything that no longer fits.

## 1. Goals and non-goals

### Goals (v1)
- Decentralized social media. Users own their data by default, and the software only takes their input and visualizes it.
- Full default user control over what, how, where, and to whom data is shared.
- Make the pages/sites as small as possible.
- The template lives on the device, so minimal data is sent between users.
- Works over the internet, Wi-Fi, and Bluetooth.
- Edit a site offline, then send a command to post it. When the command lands, it's auto-posted.
- Encrypt data end-to-end when sharing it between users.
- Pages/sites look like old Facebook, or are at least reminiscent of it.

### Not in v1, planned for v2
- Multi-device sub-keys, which allow users to post from multiple devices at once.
- A dead man's switch: send a command over LoRa, or any other method, to take down your whole page in a matter of seconds. This will need a speed focus.
- A dead man's switch that takes the page down if you miss check-ins you've set up.
- Speed. It isn't a goal for now, and v2 makes it faster.
- LoRa.
- Real users. v1 is a proof of concept and a test of myself.

### Maybe later
- Let users choose which encryption they want.

### Still open
- What "small" means in numbers. Idea: N times smaller than the same data as JSON. Decide after the first benchmark.

## 2. Identity

- **Key type:** Ed25519. Public key is 32 B and signature is 64 B. It's small, fast, and the same input always gives the same signature, so no random number is needed.
- **Identity:** the full public key. Nothing else counts as identity.
- **Fingerprint:** the first 8 B of a hash of the public key. It's only a lookup hint in headers, so a receiver knows which stored key to check. It never verifies anything.
- **Names:** every user picks a label (username) and has a short code from their key (like night#a3f9). Labels aren't unique, the key is. The code is only shown when two names clash on a device. It's only 16 bits, so it tells honest users apart and is not security.
- **Petnames:** a name you give a key on your own device. They're never global, because there's no central list to keep names unique.
- **Name changes:** if a followed name suddenly has a different key, the app warns the user. Keys are also exchanged in person by QR where possible. That's the real protection against fake names.
- **Collisions:** 8 B is 64 bits. A random clash across all users only becomes likely near 4 billion users, and a device only compares its own contacts. Faking a fingerprint on purpose takes about 2^64 tries, which is why it never verifies anything.
- **Same fingerprint, different keys:** the app compares full keys, treats them as two people, and warns.
- **Recovery:** the app generates a random 12-word phrase (128 bits) and derives the master key from it. The user never picks the words, so two people getting the same phrase is practically impossible, and no central list is needed. The word list size and the 128 bits need checking before this is locked in.
- **Login:** there is no server account. Entering the key (or the phrase that rebuilds it) loads the identity into the app, and the app signs posts with it. To move to a new device, scan a QR code from the old one or enter the phrase, then stop posting from the old device. In v1 only one device posts at a time.
- **Lost keys:** whoever has the key or phrase is the user, and nobody can reset it. If both are lost, the page is gone.

## 3. Bundle header

| Field | Size | Why |
| --- | --- | --- |
| Version + flags | 1 B | 4 bits version, 4 bits flags (one is "encrypted"). An unknown version is rejected with "update the app". |
| Network ID | 2 B | 0 = silnet 0. 65,536 networks is plenty. |
| Author fingerprint | 8 B | Says which stored key to verify against. |
| Device index | 1 B | Always 0 in v1. Reserved for v2 multi-device. |
| Sequence number | ~2 B | Only goes up. Blocks replaying an old bundle. Variable length, so it starts small and grows to 3 B past about 16,000 posts. |
| Root hash | 16 B | Hash of the content (profile and posts), cut to 16 B. Safe because the signature covers it. |
| Signature | 64 B | Ed25519 over every field above. |

- **Total:** about 94 B (estimate).
- **Signature covers:** all header fields plus the root hash, and through the root hash, the content. If a field were left out, an attacker could change it without breaking the signature.
- **Root hash and fingerprint:** both stay in v1. Drop the root hash later (-16 B) if the benchmark says so.

## 4. Profile

| Field | Limit | Encoding | Size |
| --- | --- | --- | --- |
| Username | 16 chars | 6-bit charset (a-z, 0-9, _ and .) | ~13 B |
| Nickname | 24 chars | UTF-8 text (spaces and emoji allowed) | ~10 to 25 B |
| Bio | 80 chars | dictionary-compressed text | ~50 B |
| Theme | 3 colors | background, accent, text; 1 B each in 3-3-2 RGB | 3 B |
| Profile picture | 16x16 | 2 bits per pixel, 4 colors | 64 B |
| Created time | n/a | minutes since 2026-01-01 | 3 B |

- **Total:** about 145 B (estimate).
- **Username vs nickname:** the username is the handle (@night). The nickname is the display name. Neither is unique, the key is.
- **Profile picture colors:** not stored. The 4 colors are a ramp from the background color to the accent color.
- **Fallback picture:** if a profile has no picture, the renderer draws an identicon from the key (0 B).
- **Created time:** set once on the first publish and never editable.

## 5. Posts

| Field | Limit | Encoding | Size |
| --- | --- | --- | --- |
| Time offset | n/a | minutes after the profile's created time | 3 B |
| Title (optional) | 30 chars | compressed text | 1 B + ~5 bits per char |
| Text | 100 chars | compressed text | 1 B length + ~5 bits per char |

- **Per post:** about 4 B plus the title and text (estimate).
- **Post list:** append-only. A new page starts with an empty list (1 B), and the renderer shows the default post.
- **Order:** posts are ordered by sequence number, not by time.

## 6. Rendering

- **Local template:** the look lives on the device and costs 0 B on the wire.
- **v1 look:** old Facebook: top bar in the accent color, picture column on the left, nickname as the heading, bio under it, then a "Wall" with the posts.
- **Derived colors:** dimmed text, borders, and tinted headers are mixed from the 3 stored colors by the renderer.
- **Default post:** while the post list is empty, the renderer shows "Default post" / "add a post maybe?" with the created time. It disappears with the first real post.
- **Template choice:** open question (see section 14).

## 7. Time

- **Stored time:** the author's device clock. The signature stops later changes, but it can't prove the clock was right.
- **Shown time:** the earliest valid gateway receipt if one exists. Otherwise the device time.
- **Labels:** "device time" or "via net0", so viewers can tell a claim from a receipt.
- **Receipt:** author fingerprint, sequence number, root hash, time, and gateway signature. About 76 to 100 B, depending on how much it repeats (estimate).
- **Limits:** online time is per update, not per post. A post written offline shows the time it went live. A gateway could backdate a receipt, so use 2 or more independent gateways later.
- **Sanity flags:** the renderer flags posts dated in the future or before the created time.

## 8. Encryption and access (draft)

- **Fixed per version:** v1 uses one set of algorithms. No user choice. New algorithms arrive through the version field.
- **Library:** libsodium through PyNaCl. The exact function names get confirmed during implementation.
- **Content key:** a random key per page encrypts the profile and posts.
- **Header stays readable:** relays and gateways need the header to verify and route. They never see the content.
- **Access in v1:** the author shares the page in three ways, and all three carry the same thing (the author's public key plus the content key):
  - **QR code:** the best choice in person, since it never travels over a network.
  - **Link:** opens the app and adds the page. The format is open (see section 14).
  - **Plain key:** the same data as text, to copy and paste.
- **Treat the shared data as secret:** anyone who gets the QR code, link, or key can read the page. Links can leak through chat history and link previews, so QR in person is the safest.
- **Size of the shared data:** about 64 B (32 B public key plus 32 B content key), which is roughly 88 characters as text (estimate).
- **Removing someone:** make a new content key and give it to everyone who stays. This only protects posts from then on. People who already downloaded old posts keep them.
- **Open:** per-viewer wrapped keys, so one person can be removed without re-keying everyone (see section 14).

## 9. Sync (draft)

- **Pull-based:** a device asks "everything since sequence N". No flood broadcasting.
- **Newest first:** send the profile and the latest posts first, then older posts on request.
- **Render immediately:** show whatever has arrived and show progress. Never lock the screen until a sync finishes.
- **Dedupe:** by hash.
- **Offline edits:** edits are queued. When a link is available, a post command publishes them.

## 10. Transports (draft)

- **v1:** the internet, Wi-Fi, and Bluetooth.
- **Adapter interface:** every transport sends bytes, receives bytes, and reports its maximum packet size. Anything larger than that gets fragmented.
- **Later:** LoRa, possibly starting as a channel for short signed commands (publish, takedown, check-in).

## 11. Pointer index, gateways, and discovery (draft)

- **Pointer update:** the bundle header: author, sequence number, root hash, and signature (~94 B). The content is fetched by hash.
- **One writer per page:** the highest sequence number from the right key wins. No global consensus.
- **Gateway:** any node with internet that republishes updates and issues receipts. It can delay or withhold, but not forge.
- **Abuse:** gateways rate-limit per key and reject bad signatures.
- **Intro silsite:** a welcome page that teaches silsites by being one. Bundled in the app, so it works with no network.
- **Directory:** opt-in. A small signed listing (key, name, short blurb) on the index. It lists pointers only, so nothing is hosted centrally. Clients can add other directories.

## 12. Takedown (draft)

- **v1:** a signed tombstone that compliant nodes honor. It's a request, not a guarantee, because anyone with a copy can ignore it.
- **Real deletion:** only rotating the content key makes cached copies unreadable.
- **Dead man's switch:** v2. See section 13.

## 13. Reserved for v2

- **Device index:** 1 B in the header, always 0 in v1.
- **Delegation certificates:** a reserved section for multi-device sub-keys (device public key, device index, expiry, signature from the master key).
- **Dead man's switch:** an instant takedown command and automatic takedown on missed check-ins. Policy lives in the manifest.
- **Other v2 items:** speed work and LoRa.

## 14. Versioning, limits, and open questions

- **Unknown version:** rejected with "update the app".
- **Same version:** the schema is fixed, and a new field means a new version.
- **Max bundle size:** open. Decide after the benchmark.
- **Post editing and deleting:** open. v1 option: delete a post with a signed per-post tombstone.
- **Template:** does the author store a 1 B template ID, or does each viewer pick the look?
- **Color:** is 3-3-2 good enough, or is RGB565 (+3 B) worth it?
- **Created vs published time:** the same moment, or separate fields?
- **Access:** per-viewer wrapped keys or the shared content key?
- **Link format:** a custom link that opens the app, or a web link? If it's a web link, put the keys after the `#` so they aren't sent to a server (to be confirmed).
- **Header size:** keep or drop the root hash and fingerprint (~71 B header).
