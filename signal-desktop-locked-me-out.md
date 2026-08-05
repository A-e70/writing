# My messages were on my own disk and I could not read them

Signal Desktop stopped working for me in a way I had not seen before. The window opened. My conversations were there. I could scroll years back and look at every photo. But every button that needed the network did nothing, and the screen offering to fix it hung the moment I touched it.

The log said what had happened in one line:

```
DeviceDelinked: device was deregistered
```

Signal Desktop is not an account. It is a linked device, and the server had unlinked it. On seeing that, the app did the correct thing and cleared its own credentials:

```
unlinkAndDisconnect: Client is no longer authorized; deleting local configuration
```

The official fix is to link the device again. It is also the thing you least want to do, because a fresh link wipes the local database and resyncs from scratch. My phone had a dead touchscreen, so scanning a QR code was not happening either.

That left a specific and slightly absurd situation. Several hundred messages and 273 photos were sitting on an SSD I own, in a house I live in, and the only program that could read them was the one that had stopped cooperating. The recommended repair would delete them.

They were not really stuck. Here is how to get them out.

## Three locks, not one

Signal Desktop encrypts at two levels, and the key to the second is inside the first.

The messages live in a SQLCipher database at `sql/db.sqlite`. Attachments are not in it. Each one is a separate file under `attachments.noindex`, individually encrypted, with its key stored as a row in the database it is not inside.

So you need the database key to get the attachment keys, and the database key is itself wrapped by your operating system's keyring.

## Lock one: the keyring

Open `config.json` in the Signal profile and you find something like this:

```json
{
  "encryptedKey": "763131...",
  "safeStorageBackend": "kwallet6"
}
```

That is Electron's `safeStorage`, which is Chromium's, which means the format is documented by anyone who has ever written a Chrome cookie extractor. The first three bytes are a version tag and they tell you what you are dealing with:

- `v10` means no OS keyring was available, so Chromium fell back to a hardcoded password: the string `peanuts`. If you see this, the key is effectively in plaintext.
- `v11` means a real keyring holds the password. kwallet on KDE, libsecret on GNOME.

Mine was `v11`, so the password came from the wallet:

```bash
kwallet-query -f "Chromium Keys" -r "Chromium Safe Storage" kdewallet
```

Note the folder name. Signal does not store this under its own name, it uses Chromium's, because that is the code it inherited.

## Lock two: the database

Whichever password you end up with, the unwrapping is identical, and the constants are odd enough to be worth stating plainly:

```python
aes_key = pbkdf2_hmac("sha1", password, b"saltysalt", 1, 16)
cipher  = Cipher(algorithms.AES(aes_key), modes.CBC(b" " * 16))
```

PBKDF2 with **one** iteration. A salt that is the literal string `saltysalt`. An initialisation vector of sixteen space characters. None of this is a weakness in Signal, because the real protection is the keyring holding the password. But it is a reminder that "encrypted" describes a mechanism, not a difficulty.

Out comes 64 hexadecimal characters, and that opens the database:

```python
cur.execute(f"PRAGMA key = \"x'{key}'\"")
cur.execute("PRAGMA cipher_compatibility = 4")
```

The `x'...'` syntax matters. Without it SQLCipher treats the string as a passphrase to derive a key from, rather than as the key itself, and you get the same unhelpful "file is not a database" you would get from a wrong password.

## Lock three: every attachment separately

Now the interesting part. Each file under `attachments.noindex` is laid out like this:

```
IV (16 bytes) || ciphertext || HMAC-SHA256 tag (32 bytes)
```

The per-file key is 64 bytes of base64 in the `message_attachments` table. The first 32 bytes are the AES-256-CBC key and the last 32 are the MAC key. The tag covers everything before it, so you can verify a file is intact before trusting a single byte of it.

One detail cost me a while. CBC padding hides the true length, and you cannot simply strip the padding and trust the result, because a file whose final byte happens to be `0x01` looks like it has one byte of padding whether or not it does. The database records the real plaintext size. Use it.

```python
if data and 1 <= data[-1] <= 16:
    data = data[: -data[-1]]
if plaintext_size and len(data) >= plaintext_size:
    data = data[:plaintext_size]
```

That was enough. 902 files came out: 273 photos and videos at full size, plus thumbnails, link previews and stickers.

## The part that actually mattered

Here is where I nearly made a mistake, and where I think the real lesson is.

The first run finished and printed a cheerful summary, with one line near the bottom:

```
note: 9 attachments referenced but no longer on disk
```

Nine files gone. I could have shrugged at that. Ninety-eight percent recovered is a good number, and the tool had done what I asked.

But "nine files missing" is not information. It is the absence of information wearing a number. Nine missing photos and nine missing nothing-much are completely different outcomes, and the summary could not tell me which one I was living in.

So I asked the database what those nine actually were:

```
contentType          | kind         | date
text/x-signal-plain  | long-message | 2026-06-13
text/x-signal-plain  | long-message | 2026-06-11
text/x-signal-plain  | long-message | 2026-06-14
...
```

All nine were `long-message` files, the overflow storage Signal uses when a message is too long to fit in its database row. Not one photograph among them. The true statement was not "nine files lost." It was "no pictures lost, nine long messages shortened."

That reframing took about ten minutes and changed the result from an anxious guess into something I could state as fact.

I rewrote the tool to make that distinction permanently, splitting unrecoverable data into three categories instead of one count:

**Never downloaded.** Rows with no path at all. The attachment existed in the conversation, but this device never fetched it, so it only ever lived on the phone and the server. Eighteen media items in my case, and nothing on my disk could bring them back.

**Referenced but gone.** The database points at a file that is not there, usually because Signal cleaned it up. When this happens to a picture, a thumbnail often survives, so the tool now writes the thumbnail with `_THUMBNAIL_ONLY` in the name. A small copy beats nothing.

**Orphaned.** Encrypted files on disk with no database row pointing at them. No row means no key, and without the key they are indistinguishable from noise.

## Verifying instead of hoping

Every attachment carries an HMAC, so correctness is checkable rather than assumed. But passing its own MAC only proves the decryption matched the tag. It does not prove the output is a usable file.

So after extraction I re-decoded everything independently, with tools that had no idea how it had been produced:

```bash
identify photo.jpg      # ImageMagick parses the whole image
ffprobe clip.mp4        # demuxes and reads the duration
```

851 images and 36 audio and video files, zero failures. That is a different and stronger claim than "the checksums matched."

## The lesson I did not expect

I set out to recover files. What I actually learned was about reporting.

A recovery tool that says "902 files recovered" is telling you about its own success. A recovery tool that says "902 recovered, 18 never on this machine, 9 long messages shortened, 6 orphaned without keys" is telling you about your data. The second is harder to write and it is the only one worth trusting, because it tells you what it does not know.

Any process that can partially fail should report its failures in categories, not as a single number. The number is the part that sounds reassuring, and it is the part that hides the thing you needed to hear.

---

The tool is at [github.com/A-e70/signal-desktop-export](https://github.com/A-e70/signal-desktop-export), MIT licensed. It reads data you already have, on a machine you control, using a key your own keyring already holds. Without that keyring password it does nothing at all, which is exactly as it should be.
