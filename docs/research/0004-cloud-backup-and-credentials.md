# Cloud backup target and on-device credential handling

Research for issue #4. Written 2026-08-02 against ESP-IDF v6.0 and vendor documentation
current on that date.

## The question

**Where do backups go, and how does the device hold the credential?**

The contract is fixed and not reopened here: write-mostly, append-only, no read path on the
device, disaster recovery plus lifetime archive. A restored snapshot of a dead pet is still
dead (ADR-0003), so this is archival, not a rewind. The two open choices are the concrete
target and the auth story. Cost is near zero at this volume everywhere, so the tie-breakers
are **how little code runs on the device** and **how small the blast radius is when the
credential leaks** — which, for a desk object with a two-week pet in it, it eventually will.

## How to read the numbers

Claims tagged **measured** were produced locally on 2026-08-02 by running Espressif's own
[`gen_crt_bundle.py`](https://github.com/espressif/esp-idf/blob/v6.0/components/mbedtls/esp_crt_bundle/gen_crt_bundle.py)
over the ESP-IDF `v6.0` tag inputs, or by `openssl s_client` against the live endpoints.
Everything else is a citation to a vendor reference page. Where a claim could not be
verified from a primary source it is marked **unverified** rather than inferred.

---

## The device, and the TLS floor every candidate has to clear

The M5PaperS3 is an ESP32-S3R8 at 240 MHz with **16 MB flash and 8 MB quad PSRAM**, a
960×540 ED047TC1 e-ink panel, a BM8563 RTC, a BMI270 IMU and a GT911 touch panel
([M5Stack PaperS3 spec](https://docs.m5stack.com/en/core/PaperS3)). The headline
consequence for this ticket: **flash cost is not a real constraint.** Nothing below moves
the needle against 16 MB.

ESP-IDF gives us the whole HTTPS stack for free. `esp_http_client` supports PUT and POST
with a body, arbitrary request headers via `esp_http_client_set_header()`, and TLS through
mbedTLS; server verification is wired up by setting the `crt_bundle_attach` function
pointer in `esp_http_client_config_t`
([esp_http_client](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/protocols/esp_http_client.html),
[esp-tls](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/protocols/esp_tls.html)).
mbedTLS is the default TLS stack and TLS 1.3 is supported.

### Certificate bundle cost — measured

The bundle does not store whole certificates. Espressif's Kconfig help states it stores
"default as well as customer specific root certificates in compressed format rather than
storing full certificate. For the root certificates the public key and the subject name
will be stored"
([mbedtls/Kconfig, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/components/mbedtls/Kconfig)).
That makes it far smaller than the PEM inputs suggest.

Generated from the `v6.0` tag (**measured**):

| Bundle | Roots | Bytes |
| --- | ---: | ---: |
| Full (`CERTIFICATE_BUNDLE_DEFAULT_FULL`) | 144 | 66,699 (~65 KiB) |
| Common (`CERTIFICATE_BUNDLE_DEFAULT_CMN`) | 40 | 15,905 (~15.5 KiB) |
| Custom: ISRG Root X1 only | 1 | 639 |
| Custom: Amazon Root CA 1 only | 1 | 361 |
| Custom: GTS Root R4 only | 1 | 201 |
| Custom: Sectigo Public Server Auth Root E46 only | 1 | 225 |

The same generation run against ESP-IDF `master` on 2026-08-02 produced 121 roots /
56,244 bytes full and 35 roots / 13,622 bytes common — Mozilla's store shrinks between
releases, so treat these as a snapshot, not a constant.

Espressif's Kconfig claims the common filter is "reducing the size with 50%, while still
having around 99% coverage"
([mbedtls/Kconfig, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/components/mbedtls/Kconfig));
the docs page phrases the same option as ~35 certificates at "around 94% absolute usage
coverage and 99% market share coverage"
([ESP x509 Certificate Bundle](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/protocols/esp_crt_bundle.html)).
The measured reduction is better than advertised: 15,905 / 66,699 = 24% of the full
bundle, not 50%.

**Verdict: use `CERTIFICATE_BUNDLE_DEFAULT_CMN`.** ~15.5 KiB of a 16 MB part is noise, and
it buys us insulation from the failure mode described next.

### Which root does each candidate actually chain to — measured

`openssl s_client`, 2026-08-02:

| Endpoint | Intermediate | Trust anchor presented |
| --- | --- | --- |
| `api.backblazeb2.com`, `s3.us-west-004.backblazeb2.com` | Let's Encrypt `YR2` | ISRG `Root YR`, cross-signed by **ISRG Root X1** |
| `s3.us-east-1.amazonaws.com` | Amazon RSA 2048 M04 | **Amazon Root CA 1** |
| `r2.cloudflarestorage.com` | Google Trust Services `WE1` | **GTS Root R4** |
| `api.github.com` | Sectigo Public Server Auth CA DV E36 | **Sectigo Public Server Auth Root E46** |

All four anchors are in the common list
([`cmn_crt_authorities.csv`, v6.0](https://github.com/espressif/esp-idf/blob/v6.0/components/mbedtls/esp_crt_bundle/cmn_crt_authorities.csv)),
so the 15.5 KiB bundle covers every candidate.

One finding is worth flagging because it is exactly the trap that single-root pinning sets.
Backblaze's chain today terminates at a certificate with subject `CN=Root YR, O=ISRG` that
is **not present in ESP-IDF's `cacrt_all.pem` at all** — the bundle carries ISRG Root X1 and
X2 only. Verification still succeeds because `Root YR` is itself issued by ISRG Root X1 and
the server ships that cross-signed copy in the handshake, so the chain walks up into a root
we do have. That works, and it will keep working right up until Let's Encrypt stops sending
the cross-sign, at which point a device pinned to a single root bricks its backup path
silently and permanently. **Do not pin a single root** on a device that must survive years
of CA rotation with no read path to tell you it has stopped working. The 15 KiB is the
insurance premium.

### RAM, not flash, is the real TLS cost — unverified

Espressif's ESP-TLS page documents `ESP_TLS_DYN_BUF_RX_STATIC` and the
`MBEDTLS_DYNAMIC_BUFFER` family of options for shrinking handshake heap
([esp-tls](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/protocols/esp_tls.html),
[mbedtls/Kconfig](https://github.com/espressif/esp-idf/blob/v6.0/components/mbedtls/Kconfig)),
but **neither page states a numeric heap or flash figure**, and the "Minimizing Binary Size"
guide gives no kilobyte measurements for mbedTLS either. I could not verify a real
handshake heap number without building and running on hardware. The board has 8 MB PSRAM,
so this is very unlikely to bite, but it is unmeasured. What is worth measuring later is
**energy per handshake**, which feeds the power budget in #12 — a candidate needing three
TLS handshakes per upload costs three times a candidate needing one.

---

## Candidate 1: Backblaze B2, native API

B2 is the only candidate whose **credential model itself** expresses "write, never read,
never delete". Its application keys carry an explicit capability list:

> listAllBucketNames, listBuckets, readBuckets, writeBuckets, readBucketEncryption,
> writeBucketEncryption, readBucketRetentions, writeBucketRetentions, listFiles, readFiles,
> shareFiles, writeFiles, deleteFiles, readFileLegalHolds, writeFileLegalHolds,
> readFileRetentions, writeFileRetentions, bypassGovernance, readBucketReplications,
> writeBucketReplications, readBucketNotifications, writeBucketNotifications,
> readBucketLogging, writeBucketLogging

([Application Keys](https://www.backblaze.com/docs/cloud-storage-application-keys))

Keys can additionally be restricted to specific buckets (`bucketIds`) and to a file-name
prefix (`namePrefix`), with an optional `validDurationInSeconds` capped just under 1000
days; the secret "is the only time it will be returned"
([b2_create_key](https://www.backblaze.com/apidocs/b2-create-key)). With `namePrefix` set,
"requests to list other files are denied" and "reading, writing, and deleting are allowed
only for matching files"
([Application Keys](https://www.backblaze.com/docs/cloud-storage-application-keys)).

The capability boundaries that matter:

- `b2_get_upload_url` — "The token must have the `writeFiles` capability"
  ([b2_get_upload_url](https://www.backblaze.com/apidocs/b2-get-upload-url)).
- `b2_delete_file_version` — "The token must have the `deleteFiles` capability"
  ([b2_delete_file_version](https://www.backblaze.com/apidocs/b2-delete-file-version)).
- `b2_hide_file` — "The token must have the `writeFiles` capability", and the operation
  "hides a file so that downloading by name will not find the file, though previous
  versions remain stored in the system"
  ([b2_hide_file](https://www.backblaze.com/apidocs/b2-hide-file)).

And B2 is versioned unconditionally: "A file has a list of versions with the newest version
listed first. When you download a file by name, you always get the most recent version, the
first one in the list"
([File Versions](https://www.backblaze.com/docs/cloud-storage-file-versions)). Re-uploading
an existing name adds a version rather than replacing content.

So a key with capabilities `["writeFiles"]`, one bucket, one prefix, **cannot read the
archive, cannot list it, and cannot destroy any byte of it.** It can append. It can also
call `b2_hide_file`, which is the one soft spot — an attacker can make objects invisible to
name lookup, but the versions survive and an operator key recovers them. That is vandalism,
not destruction.

**Device-side shape:** three requests, no request signing. `GET b2_authorize_account` with
HTTP Basic (`base64(keyId:appKey)`) returning an `authorizationToken` "valid for at most 24
hours" plus `apiUrl`
([b2_authorize_account](https://www.backblaze.com/apidocs/b2-authorize-account));
`b2_get_upload_url`, whose URL and token are "valid for 24 hours or until the endpoint
rejects an upload"
([b2_get_upload_url](https://www.backblaze.com/apidocs/b2-get-upload-url)); then
`POST` to that URL with `Authorization`, `X-Bz-File-Name` (percent-encoded UTF-8),
`Content-Type`, `Content-Length` (chunked encoding unsupported) and `X-Bz-Content-Sha1`
([b2_upload_file](https://www.backblaze.com/apidocs/b2-upload-file)).

Cost: "First 10GB storage is always free", $6.95/TB-month beyond, and "Class A, B, and C API
calls are free for pay-as-you-go customers"
([B2 pricing](https://www.backblaze.com/cloud-storage/pricing)) — uploads are free
transactions.

**Against it:** it needs a JSON parser on-device (cJSON ships with ESP-IDF), SHA-1 of the
payload, a cached-token state machine with two expiry paths, and up to three TLS handshakes
on a cold start (the auth host and the returned upload host differ). Steady state after
caching is one handshake per upload.

## Candidate 2: Amazon S3

S3 can be made append-only, but by policy authorship rather than by credential shape. The
building blocks are all real and documented:

- **Versioning** means "if you overwrite an object, it results in a new object version in
  the bucket" ([S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)),
  and "To delete versioned objects permanently, you must use `DELETE Object versionId`"
  ([Deleting object versions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DeletingObjectVersions.html))
  — which is gated by `s3:DeleteObjectVersion`, a distinct action from `s3:DeleteObject`
  ([S3 actions and condition keys](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazons3.html)).
  An IAM policy granting only `s3:PutObject` on one prefix is therefore already append-only.
- **Conditional writes** close the overwrite hole outright. `If-None-Match: *` on PutObject
  fails with `412 Precondition Failed` if the key exists, and this can be *required* by
  bucket policy via the `s3:if-none-match` condition key — AWS publishes the exact policy
  (`"Condition": {"Null": {"s3:if-none-match": "false"}}`)
  ([conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html),
  [enforcing them](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes-enforce.html)).
  Note it requires SigV4 signing.
- **Object Lock** in compliance mode is the strongest immutability guarantee any candidate
  offers: "a protected object version can't be overwritten or deleted by any user, including
  the root user in your AWS account… The only way to delete an object under the compliance
  mode before its retention date expires is to delete the associated AWS account"
  ([S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).
  It requires versioning to be enabled.

**Against it:** SigV4. Every request needs a canonical request, a SHA-256 of the payload, a
four-step HMAC-SHA256 key derivation and a correctly formatted `x-amz-date` — meaning the
backup path acquires a **hard dependency on the device clock being right**, which drags in
#18 (RTC drift and NTP) as a prerequisite for backups working at all. mbedTLS supplies the
primitives, but this is the largest device-side code surface of any candidate. I could not
verify current S3 request pricing from the pricing page (the tables did not render in
fetched form); the ticket's premise that cost is negligible here is almost certainly right
but is **unverified** in this note.

## Candidate 3: Cloudflare R2

R2 has exactly four token scopes: Admin Read & Write, Admin Read only, Object Read & Write
("read, write, and list objects in specific buckets"), and Object Read only
([R2 API tokens](https://developers.cloudflare.com/r2/api/tokens/)). **There is no
write-only or append-only scope.** The narrowest token we can give the device can read the
entire archive and delete from it. Tokens can be scoped to a set of buckets; the docs do not
document key-prefix scoping.

Access is via the S3-compatible API — the Access Key ID is "the `id` of the API token" and
the Secret is "the SHA-256 hash of the API token `value`", and "Object Read & Write and
Object Read only permissions are only supported by the S3-compatible API, not the Cloudflare
REST API" ([R2 S3 API tokens](https://developers.cloudflare.com/r2/api/s3/tokens/)) — so
using the simpler-looking REST path would force us up to an Admin token. Signing is SigV4:
Cloudflare's own example emits `X-Amz-Algorithm=AWS4-HMAC-SHA256` with `region: "auto"`
([aws4fetch example](https://developers.cloudflare.com/r2/examples/aws/aws4fetch/)).

The mitigation is server-side rather than credential-side: **bucket locks** "prevent the
deletion and overwriting of objects in an R2 bucket for a specified period — or
indefinitely", and "a bucket cannot be emptied while any bucket lock rules are configured"
([R2 bucket locks](https://developers.cloudflare.com/r2/buckets/bucket-locks/)).
Configuring them needs "an API token with permissions to edit R2 bucket configuration",
which by Cloudflare's own scope definitions is an Admin token, not the Object Read & Write
token the device would hold — so a stolen device token could not lift the lock. I could not
find an explicit statement to that effect on the bucket-locks page; it follows from the
scope descriptions but is **unverified as a direct quote**.

Pricing is generous — 10 GB-month storage and 1 million Class A ops free, then $0.015/GB-mo
and $4.50/million Class A; PutObject is Class A
([R2 pricing](https://developers.cloudflare.com/r2/pricing/)).

**Net:** identical device code to S3 (SigV4, same clock dependency), strictly worse
credential story, because the leaked key can read every snapshot the pet ever wrote.

## Candidate 4: GitHub contents API

This is the worst option on the security axis and it is not close.

`PUT /repos/{owner}/{repo}/contents/{path}` needs `message`, base64 `content`, and a `sha`
when replacing an existing file, with a 100 MB ceiling
([Repository contents](https://docs.github.com/en/rest/repos/contents)). The fine-grained
PAT permission required is **Contents: write**, and that same permission grants
`DELETE /repos/{owner}/{repo}/contents/{path}`, `PATCH /repos/{owner}/{repo}/git/refs/{ref}`,
`POST /repos/{owner}/{repo}/git/refs` and `DELETE /repos/{owner}/{repo}/git/refs/{ref}`
([fine-grained PAT permissions](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens)).
A leaked token can therefore force-update a ref and erase the entire archive, and there is
no scope below "the repository". Whether Contents:write also implies Contents:read is
**unverified** — the docs table separates the columns and I could not find a definitive
statement — but it is moot, because delete is unambiguously included.

Tokens can be set never to expire: "Infinite lifetimes are allowed but may be blocked by a
maximum lifetime policy set by your organization or enterprise owner"
([managing PATs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)).
A never-expiring, delete-capable, repo-wide credential sitting in a desk toy's flash is the
exact thing this ticket exists to avoid.

Rate limits also bite: 5,000 requests/hour, and "no more than 80 content-generating requests
per minute and no more than 500 content-generating requests per hour"
([REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)).
Fine for our cadence, but it means the archive shares a budget with anything else the
account does.

The one genuine attraction — the care log rendered as a browsable git history — is a *read*
feature, and this ticket's contract says the device has no read path. It can be had later by
mirroring from the real archive, without putting a repo-write token on the device.

**Rejected.**

## Candidate 5: a self-hosted (or serverless) append endpoint

Put a thin HTTPS receiver in front of storage — a Cloudflare Worker with an R2 binding, or a
20-line handler on a VPS — and hand the device a bearer token.

This is the **simplest possible device code**: one HTTPS PUT/POST, a static
`Authorization` header, no signing, no JSON parsing, no clock dependency, no token dance,
one host, one TLS handshake. It is also the **tightest capability model available**, because
the credential's power is bounded by the verbs the server implements. If the endpoint only
implements "append a blob under this device's prefix", then a stolen credential can only
append a blob under that device's prefix — there is no read or delete verb to abuse, no
matter what the underlying store supports. The storage credential never leaves the server.

**Against it:** it is a second thing to keep alive for the lifetime of the archive. The
whole point of "lifetime archive" is that it outlives our attention, and a Worker or VPS is
precisely the sort of thing that quietly stops being paid for. It also means the backup
path's availability is bounded by ours, not by a storage vendor's.

---

## The credential threat model

### What actually sits on the device

One long-lived secret in NVS, plus non-secret configuration (bucket id, prefix, endpoint
host). Wi-Fi credentials are already there — the ESP-IDF Wi-Fi driver "stores credentials
(like SSID and passphrase) in the default NVS partition"
([NVS Encryption](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/storage/nvs_encryption.html))
— so the device is already holding something worth protecting and the marginal decision is
smaller than it first looks.

### The realistic attackers, in order of likelihood

1. **Us.** The credential gets committed to the repo, pasted into an issue, or baked into a
   firmware image that ends up published. This is the overwhelmingly most likely leak and
   *no* on-device measure prevents it.
2. **A shared flash image.** Someone dumps the device and posts the binary, or we hand a
   built image to a friend to flash.
3. **Casual theft.** The desk object walks. The thief almost certainly wants the hardware,
   not a fortnight of Tamagotchi telemetry.
4. **A targeted attacker with the device on a bench.** Realistically out of scope for a
   single hobby unit, but it is the one the eFuse machinery is designed for.

For (1) and (2), the only defence that works is **making the credential worthless to hold**.
That is a capability-model question, not an encryption question — which is why the choice of
target dominates the choice of key storage.

### What NVS encryption actually buys

Two schemes exist. The flash-encryption-based scheme keeps XTS keys in a dedicated
`nvs_keys` partition protected by flash encryption, so "enabling Flash Encryption becomes a
prerequisite for NVS encryption here". The HMAC scheme derives the XTS keys from an HMAC key
in eFuse with purpose `ESP_EFUSE_KEY_PURPOSE_HMAC_UP`, and "since the encryption keys are
derived at runtime, they are not stored anywhere in the flash… This scheme enables us to
achieve secure storage on ESP32-S3 **without enabling flash encryption**." It consumes one
eFuse block, selected by `CONFIG_NVS_SEC_HMAC_EFUSE_KEY_ID` in range 0–5
([NVS Encryption](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/storage/nvs_encryption.html)).
Data is encrypted with XTS-AES, each entry treated as a sector.

**The HMAC scheme is the right call for this project**: it defeats the passive flash dump
(threat 2, and threat 3 for anyone short of a bench attacker) at the cost of one eFuse key
slot and no change to the flashing workflow.

Be honest about what it does *not* do. The ESP-IDF page describes the algorithm and the key
provenance but **does not state which parts of NVS are left in plaintext** — page headers,
entry-state bitmaps, namespace and key names are not addressed either way. I could not
verify from Espressif's documentation whether NVS key names leak. Assume they might, and do
not encode anything sensitive into the *name* of the NVS entry.

### What flash encryption adds, and what it costs

Flash encryption means "physical readout of flash will not be sufficient to recover most
flash contents", with the key in `BLOCK_KEYN` eFuses where it "cannot be accessed via
software as the write and read protection bits… are set". Enabling it also causes the second
stage bootloader to burn `DIS_PAD_JTAG` and `DIS_USB_JTAG`, and Release mode additionally
burns `DIS_DOWNLOAD_MANUAL_ENCRYPT` and write-protects `SPI_BOOT_CRYPT_CNT`, after which
"new plaintext images can ONLY be downloaded using the over-the-air (OTA) scheme"
([Flash Encryption](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/security/flash-encryption.html)).

Espressif's own limitations list is the part worth internalising:

> - Flash encryption is only as strong as the key…
> - Not all data is stored encrypted. If storing data on flash, check if the method you are
>   using (library, API, etc.) supports flash encryption.
> - Flash encryption does not prevent an attacker from understanding the high-level layout
>   of the flash…
> - Flash encryption alone may not prevent an attacker from modifying the firmware of the
>   device. To prevent unauthorised firmware from running on the device, use flash
>   encryption in combination with Secure Boot.

**Recommendation: don't.** Release-mode flash encryption plus Secure Boot v2 is a
one-way door on a single hobby device — it costs the serial reflash workflow forever, in
exchange for defending against threat (4), which is not our threat. And note the ceiling on
all of it: even a perfectly locked device still *uses* the credential. An attacker who
controls the running firmware can make it upload on demand regardless of how the key is
stored. Encryption at rest bounds the passive-dump case; it never makes the credential safe.

### "Write-only by construction", candidate by candidate

This is the question that actually decides the ticket. What can a leaked credential do?

| Target | Read archive | List archive | Overwrite | Permanently delete | Enforced by |
| --- | --- | --- | --- | --- | --- |
| **B2**, key = `["writeFiles"]` + bucket + prefix | No | No | No (new version) | No (`deleteFiles` withheld) | Capability list on the key |
| **S3**, IAM = `s3:PutObject` only, versioning on | No | No | No (new version) | No (`s3:DeleteObjectVersion` withheld) | IAM policy + bucket versioning |
| **S3** + Object Lock compliance | No | No | No | **No, not even by root** | Bucket configuration |
| **R2**, Object Read & Write | **Yes** | **Yes** | Yes | Yes, unless bucket lock | Bucket lock only |
| **GitHub**, Contents: write | Probably (unverified) | Yes | Yes | **Yes — ref rewrite** | Nothing |
| **Self-hosted append endpoint** | No | No | No | No | The server's verb set |

B2 and S3 both achieve the goal; S3 achieves it more strongly (compliance-mode Object Lock
is unappealable) but B2 achieves it *in the credential itself*, which is the property that
survives someone later editing an IAM policy without thinking. R2 and GitHub do not achieve
it at all.

One residual for B2: `b2_hide_file` needs only `writeFiles`, so a leaked key can hide files
from name lookup — but "previous versions remain stored in the system"
([b2_hide_file](https://www.backblaze.com/apidocs/b2-hide-file)), so nothing is lost and an
operator key undoes it. Cost of the mitigation: none available at the capability level;
accept it.

### Rotation and revocation

Whatever we pick, the operator-side story is the same and should be written down at
provisioning time: the credential is revocable from the vendor console, the device tolerates
`401` by failing silently and retrying (below), and re-provisioning is a Wi-Fi-provisioning
flow (#19) rather than a reflash. B2 keys additionally support `validDurationInSeconds` up
to just under 1000 days ([b2_create_key](https://www.backblaze.com/apidocs/b2-create-key)),
which is a usable dead-man's switch if we want the credential to expire rather than live
forever — at the cost of the archive silently stopping if nobody renews it. Given that the
device has no read path and therefore no way to *tell* us backups stopped, a hard expiry is
a footgun. Prefer a non-expiring key that is easy to revoke.

---

## Snapshot cadence and naming

Append-only plus latest-only restore makes this simple, and the constraint from ADR-0003 is
the binding one: "the save schema must retain mistakes as **dated events**, not as an
integer". The archive must therefore carry the care log, not just the current meters.

**Naming.** Every object is immutable and the key encodes its own ordering, so no read path
is needed to compute the next name:

```
pets/<hatch-instant>/<zero-padded-seq>-<unix-seconds>.<ext>
pets/<hatch-instant>/final.<ext>          # written once, at death
```

The hatch instant is the pet's identity (`CONTEXT.md`: "All ages are measured from it, which
keeps age immune to timezone and DST"), which gives the graveyard a natural directory per
pet without the device needing to enumerate anything. The sequence number is the device's
local monotonic save counter and comes from the persistence layer (#5), so restore picks the
lexicographically last key without parsing timestamps, and a wrong wall clock cannot reorder
history. Keep the Unix seconds in the name anyway for human legibility in a bucket listing.

**Cadence.** A full snapshot every time, not deltas — deltas would require reading the
archive to reconstruct, and we have deliberately given the device no read path. Trigger on
*state-changing events* (care events and care mistakes, per `CONTEXT.md`), coalesced to at
most one upload per interval, plus a forced snapshot per Year (the 24-hour age unit) so that
a quiet pet still leaves a trail. At 2–4 care events per day plus a daily floor, that is on
the order of 5 uploads/day and roughly 70 objects per two-week life.

Sizing is **unverified** until #5 settles the serialisation, but the arithmetic is
insensitive: at a generous 8 KiB per snapshot, one pet life is ~0.6 MB and a decade of
back-to-back pets is well under 200 MB — inside B2's permanently-free 10 GB and inside R2's
free tier ([B2 pricing](https://www.backblaze.com/cloud-storage/pricing),
[R2 pricing](https://developers.cloudflare.com/r2/pricing/)), with B2's Class A uploads free
outright.

If upload frequency turns out to cost real battery (#12), the lever is the coalescing
interval, not the snapshot format.

## Behaviour when the network is down

Non-negotiable: **the game is fully playable with no network, and backup failure is never
surfaced.** A pet that cannot be fed because Wi-Fi is down would be a worse bug than losing
the archive.

- **Backup never blocks the care loop.** Snapshots are handed to a queue; the game loop's
  only interaction with the network is enqueuing bytes.
- **Queue on local storage.** The pending snapshot lives in the same crash-safe store #5
  chooses. Keep a bounded queue — a small ring of the most recent snapshots plus a hard-kept
  `final` object — and drop oldest-first when full. Losing intermediate snapshots is
  acceptable; the contract is latest-only restore.
- **Retry with backoff, silently.** Wi-Fi down, DNS failure, TLS failure, `401`, `503` are
  all the same event to the game: nothing happened. No care log line, no icon, no buzzer.
  The care log is the player-facing record of *care* (`CONTEXT.md`), not of plumbing.
- **Retries are safe because keys are deterministic.** The same snapshot always maps to the
  same key. On S3 with enforced conditional writes, a duplicate retry returns `412
  Precondition Failed`
  ([conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html))
  — treat 412 as success, not as an error. On B2 a duplicate retry adds a harmless extra
  version ([File Versions](https://www.backblaze.com/docs/cloud-storage-file-versions)).
- **Handle token expiry as an ordinary retry.** B2's auth token is "valid for at most 24
  hours" ([b2_authorize_account](https://www.backblaze.com/apidocs/b2-authorize-account))
  and the upload URL for "24 hours or until the endpoint rejects an upload"
  ([b2_get_upload_url](https://www.backblaze.com/apidocs/b2-get-upload-url)), so the state
  machine needs exactly one rule: on any auth failure, discard cached tokens and re-run the
  three-step dance on the next attempt.
- **Opportunistic scheduling.** Flush the queue when the device is awake anyway — during a
  Session, or on the wake that a forced snapshot triggers — rather than waking the radio on
  its own schedule. This is the cheapest lever on #12.

---

## Recommendation

**Target: Backblaze B2 via the native API. Credential: a single application key with
capabilities `["writeFiles"]`, restricted to one bucket and one `namePrefix`, stored in NVS
with the HMAC-eFuse encryption scheme. No flash encryption, no Secure Boot. Certificate
verification via `esp_crt_bundle` with the common bundle.**

The deciding argument is not cost, code size, or flash. It is that B2 is the only
zero-infrastructure candidate where **"write-only" is a property of the credential rather
than of a policy document someone has to keep correct.** The device physically holds a
secret that cannot read the archive, cannot list it, and cannot delete a byte of it. If it
leaks into a git commit tomorrow, the worst outcome is a stranger appending junk under one
prefix of one bucket, and the archive itself is intact and recoverable. That is the smallest
blast radius available without running a server, and it is the thing issue #4 actually asks
for.

Secondary arguments: no request signing, so no dependency on the device clock being correct
for backups to work — which decouples this from #18 in a way S3 and R2 do not; free uploads
and a permanently free 10 GB tier that this workload will never exhaust; and ~15.5 KiB of
certificate bundle against 16 MB of flash.

### Honest trade-offs

- **B2's device code is not the simplest.** It needs cJSON, SHA-1, and a two-token cache
  with expiry handling, versus S3's single signed PUT. I judge the SigV4 canonical-request
  machinery to be the larger and more error-prone body of code, and the clock dependency to
  be the more dangerous coupling — but this is a judgement call, not a measurement, and
  someone who has already written SigV4 could reasonably rank them the other way.
- **We give up compliance-mode Object Lock.** S3 offers a guarantee B2's capability model
  does not: an archive that "can't be overwritten or deleted by any user, including the root
  user"
  ([S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)).
  If the archive's durability against *our own* mistakes matters more than device-side
  simplicity, S3 with versioning, `s3:PutObject`-only IAM, enforced conditional writes and
  compliance-mode Object Lock is the stronger answer, and it is a defensible reversal.
- **We give up the browsable-history charm of GitHub.** Recoverable later by mirroring from
  B2, without ever putting a repo-write token on the device.
- **Cold start costs up to three TLS handshakes.** Steady state is one; unmeasured in energy
  terms, and it is the first thing to check against #12.
- **`b2_hide_file` remains reachable by the device key.** Accepted: it hides, it does not
  destroy.
- **B2 is one vendor, and this is a lifetime archive.** Every candidate has this problem.
  The mitigation is that the object naming above is vendor-neutral and a bulk copy out is a
  single `rclone` invocation from an operator machine.

**The named alternative, if we are willing to run a shim:** a Cloudflare Worker with an R2
binding, device holding a bearer token. Strictly better on both device simplicity (one
unsigned PUT, one handshake, no clock, no JSON) and credential blast radius (the credential
can do exactly what the Worker implements and nothing else). Rejected as the primary only
because it makes the archive's survival depend on a service we have to keep alive for
years, which is the one thing a lifetime archive must not do. If a shim already exists for
other reasons, prefer it.

---

## What remains uncertain

- **mbedTLS heap and flash cost on this target.** Espressif documents the tuning knobs but
  publishes no kilobyte or heap figures I could find, and the "Minimizing Binary Size" guide
  gives none. Unmeasured until we build and run.
- **Energy per TLS handshake**, and therefore whether cold-start cost or upload cadence
  matters to the one-week-per-charge target in #12. Unmeasured.
- **What NVS leaves in plaintext.** The ESP-IDF NVS encryption page describes XTS-AES over
  entries but makes no statement about page headers, entry-state bitmaps, namespace names or
  key names. Treat NVS key names as potentially visible.
- **Snapshot size**, which depends entirely on the serialisation #5 picks. The cadence
  arithmetic above is insensitive to it across any plausible range, but the number is a
  guess.
- **Whether an R2 Object Read & Write token can alter bucket lock configuration.** It
  follows from Cloudflare's scope definitions that it cannot (editing bucket configuration
  is listed only under Admin Read & Write), but the bucket-locks page says only "an API
  token with permissions to edit R2 bucket configuration" and does not name the scope. Not
  load-bearing for the recommendation.
- **Whether GitHub's Contents: write implies Contents: read.** The permissions table
  separates the columns and I found no explicit statement. Moot — delete is unambiguously
  included, which is enough to reject the option.
- **Current AWS S3 request pricing.** The pricing page's tables did not render in fetched
  form. Assumed negligible per the ticket's premise; not verified here.
- **B2's `b2_upload_file` capability requirement is inferred, not quoted.** The upload page
  itself does not name a capability; `writeFiles` is documented as required for
  `b2_get_upload_url`, which gates the upload. The `X-Bz-Content-Sha1` alternatives
  (`do_not_verify`, hex-digits-at-end) are referenced obliquely by the docs but I could not
  quote a definitive statement of the accepted values.
- **CA rotation risk over the device's life.** The Backblaze chain currently validates only
  because Let's Encrypt ships a cross-signed `Root YR` up into ISRG Root X1, which *is* in
  the bundle; `Root YR` itself is not. The common bundle absorbs this today. Nothing
  guarantees it absorbs the next rotation, and a device with no read path cannot tell us
  when it stops working. Worth a deliberate decision later about how the certificate bundle
  gets refreshed — probably "it ships with firmware updates", which means backups depend on
  the OTA story existing.
