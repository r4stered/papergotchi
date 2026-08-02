# Persistence: NVS, LittleFS or SD — and crash-safe saves

Research note for [#5](https://github.com/r4stered/papergotchi/issues/5). Investigated 2026-08-02
against ESP-IDF v6.0 documentation, the `esp-idf` sources on `master`, the upstream littlefs
repository, and the M5Stack and Winbond datasheets. Every factual claim below carries a URL; where
a claim could not be verified from a primary source it is marked as such rather than inferred.

## The question

**NVS, LittleFS, or the SD card — and how do we guarantee a save is never half-written?**

The pet lives roughly two weeks and death is permanent. The **hatch instant** anchors every age the
game computes and must survive power loss. The **care log** is a growing timestamped record and
ADR-0003 requires care mistakes to persist as dated events rather than as a counter. The graveyard
is permanent. The battery model makes brownouts an expected event, not an edge case. A torn write
that destroys a fortnight-old pet is the worst bug this project can ship.

---

## Finding 0: the premise "state changes every few minutes" is not the same as "state must be saved every few minutes"

This deserves stating before any store is compared, because it moves the answer more than the
choice of store does.

Meters are a deterministic function of elapsed wall-clock time since an anchored state. The device
already has to do this: the map's flat-battery decision says "unpowered time accrues neglect", which
means the core must be able to fast-forward from an old snapshot to now. If `step(state, elapsed,
seed)` is a pure function — which the map's core/port split already demands — then **a save is only
needed when something happens that a replay could not reproduce**:

- a **care event** (2–4 per day, per the map's life clock)
- a committed random outcome (illness onset, a **needless call**), unless it is derived
  deterministically from a seed stored in the save
- a stage boundary, an evolution, entering the **rescue window**, death
- a coarse heartbeat to bound the replay window and to survive RTC weirdness

That is roughly 10–30 writes per day, not 288. The endurance arithmetic below is presented for both
the event-driven rate and the ticket's stated "every few minutes" rate, so the conclusion holds
either way — but the event-driven rate is what should be built.

**This imposes a hard requirement on the core**: any randomness must be reproducible from state the
save contains. Otherwise rolling back to a previous slot and replaying produces a different pet.
See "Recovery" below.

---

## Finding 1: NVS

### What it is, and how it lays out flash

NVS stores key-value pairs in a partition of type `data` subtype `nvs`, reached through the
`esp_partition` API.[^nvs-doc] Its internal structure is documented in full, which is unusual and
useful — we can reason about atomicity from the on-flash format rather than from marketing prose.

- One **logical page = one 4096-byte flash sector**. A page is a 32-byte header, a 32-byte entry
  state bitmap, and **126 entries of 32 bytes each** (32 + 32 + 126×32 = 4096).[^nvs-doc]
- Writes are **append-only**: "NVS stores key-value pairs sequentially, with new key-value pairs
  being added at the end. When a value of any given key has to be updated, a new key-value pair is
  added at the end of the log and the old key-value pair is marked as erased."[^nvs-doc]
- Each entry carries a **CRC32** "calculated over all the bytes in this entry, except for the CRC32
  field itself", and string/blob data carries a second CRC32 "calculated over all bytes of
  data".[^nvs-doc]
- Each entry has a 2-bit state in the bitmap: `Empty (2'b11)`, `Written (2'b10)`, `Erased
  (2'b00)`.[^nvs-doc] Note that every transition is a 1→0 bit flip, so state changes need no erase.
- Pages carry a sequence number and a state (`Empty`, `Active`, `Full`, `Erasing`, `Corrupted`).
  Of `Erasing`: "This is a transient state… In case of a sudden power off, the move-and-erase
  process will be completed upon the next power-on."[^nvs-doc]

### What Espressif actually says about power loss — read this exactly

> "The library does try to recover from conditions when flash memory is in an inconsistent state. In
> particular, one should be able to power off the device at any point and time and then power it
> back on. This should not result in loss of data, except for the new key-value pair if it was being
> written at the moment of powering off. The library should also be able to initialize properly with
> any random data present in flash memory."[^nvs-doc]

Read the modality: *"does try to recover"*, *"should not result"*, *"should also be able"*. This is
a statement of design intent and tested behaviour. **It is not a formal guarantee, and this note
will not restate it as one.** It is, however, the strongest such statement Espressif makes about any
of its storage components, and it is backed by tests (below).

### Verifying the claim in source

The claim is checkable, and it checks out. All paths below are on `espressif/esp-idf` `master`.

**Per-entry torn-write detection.** `Page::writeEntry()` writes the entry data first and *then*
flips the bitmap to `WRITTEN`:[^nvs-page]

```cpp
err = mPartition->write(phyAddr, &item, sizeof(item));
...
err = alterEntryState(mNextFreeEntry, EntryState::WRITTEN);
```

On load, the same file detects the resulting hole explicitly — the comment is worth quoting because
it names the exact failure mode this ticket is about:

```cpp
// however, if power failed after some data was written into the entry.
// but before the entry state table was altered, the entry locacted via
// entry state table may actually be half-written.
// this is easy to check by reading EntryHeader (i.e. first word)
```

…and any entry whose first word is not `0xffffffff` but whose state is not `WRITTEN` is forced to
`ERASED`.[^nvs-page] A half-written entry can therefore never be read back as data.

**Blobs are already double-buffered with a commit record.** This is the important one.
`Storage::writeMultiPageBlob()` writes all `BLOB_DATA` chunks first and writes the `BLOB_IDX` entry
— the record that makes the blob findable — **last**.[^nvs-storage] And `Storage::writeItem()` for a
`BLOB` reads the previous index's `chunkStart`, then **toggles it** so the new version's chunks
cannot land on the old version's:[^nvs-storage]

```cpp
prevStart = item.blobIndex.chunkStart;
// Toggle the version by changing the offset
nextStart = (prevStart == VerOffset::VER_1_OFFSET) ? VerOffset::VER_0_OFFSET : VerOffset::VER_1_OFFSET;
err = writeMultiPageBlob(nsIndex, key, data, dataSize, nextStart, purgeAfterErase);
```

Only after that succeeds is the previous version erased (`eraseMultiPageBlob(..., prevStart)`).
On mount, `Storage::init()` calls `eraseMismatchedBlobIndexes()`, which sums the observed chunk
sizes and counts against what each `BLOB_IDX` claims and deletes any index that does not add up,
then `eraseOrphanDataBlobs()` sweeps the chunks left behind.[^nvs-storage]

**In other words, NVS already implements the scheme the ticket proposes** — versioned slots, a
commit record written last, CRC-checked, with a mount-time consistency sweep — for a single blob
under a single key. We do not have to build it. What we do have to build is described in Finding 5.

**Duplicate resolution favours the new value.** If power is lost after the new value is written but
before the old is erased, `PageManager::load()` finds the last item on the highest-sequence page and
erases the older duplicate — the *newer* value wins.[^nvs-pagemgr] Consequence: after a *successful*
save there is exactly one recoverable generation. NVS's internal double-buffering does not give us a
previous-generation fallback.

**Espressif fuzz-tests this.** `test_nvs.cpp` contains `TEST_CASE("test recovery from sudden
poweroff", "[long][nvs][recovery][monkey][.]")`, which drives random NVS operations while stepping a
power-off injection point, plus about a dozen targeted cases ("Recovery from power-off when page is
being freed", "recovery after failure to write data", …).[^nvs-test] This is the best evidence
available that the hedged prose above is meant seriously.

### `nvs_commit` is a no-op — the write is already through

The API docs say "the actual storage will not be updated until `nvs_commit` is called".[^nvs-doc]
The implementation disagrees:[^nvs-api]

```cpp
extern "C" esp_err_t nvs_commit(nvs_handle_t c_handle)
{
    Lock lock;
    // no-op for now, to be used when intermediate cache is added
```

So `nvs_set_blob()` returning `ESP_OK` means the bytes are on flash today. **Call `nvs_commit()`
anyway** — it costs nothing and keeps us correct if Espressif ever adds the cache the comment
anticipates. Do not build any logic that depends on either reading.

### Wear levelling

NVS's append-only log *is* its wear levelling: "The organization of NVS space to pages and entries
effectively reduces the frequency of flash erase to flash write operations by a factor of
126."[^nvs-doc] Free pages are managed FIFO — `mFreePageList.pop_front()` in `activatePage()`,
`push_back()` when a page is released[^nvs-pagemgr] — so erases round-robin across the partition's
sectors rather than concentrating.

This is *dynamic* wear levelling scoped to the partition. It does not move static data, and it does
not level across the rest of the flash chip. Sizing the partition is therefore how we buy endurance
headroom (see Finding 6).

### Costs to know about

- Mount time: "Initialization of NVS requires approximately 0.5 seconds per 1000 keys."[^nvs-doc]
  At ~80 keys that is ~40 ms per cold boot. On a device that deep-sleeps between RTC wakes this is a
  per-wake CPU cost and belongs in the power budget conversation (#12).
- RAM: "each 1 MB of NVS flash partition consumes 22 KB of RAM and each 1000 keys consumes 5.5 KB of
  RAM."[^nvs-doc] A 64 KB partition with 80 keys is well under 2 KB.
- Blob ceiling: "Blob values are limited to 508,000 bytes or 97.6% of the partition size − 4000
  bytes, whichever is lower."[^nvs-doc] Not a constraint for us.
- Minimum partition: "at least 0x3000 bytes" (3 pages, one of which must stay `Empty`).[^parttab]
- NVS "works best for storing many small values, rather than a few large values of the type 'string'
  and 'blob'."[^nvs-doc] Our records are small. Good fit.

---

## Finding 2: LittleFS

LittleFS makes the strongest and clearest power-loss claims of anything considered here, and states
them in its own README:

> "**Power-loss resilience** - littlefs is designed to handle random power failures. All file
> operations have strong copy-on-write guarantees and if power is lost the filesystem will fall back
> to the last known good state."[^lfs-readme]

> "All POSIX operations, such as remove and rename, are atomic, even in event of power-loss.
> Additionally, file updates are not actually committed to the filesystem until sync or close is
> called on the file."[^lfs-readme]

The mechanism is documented in SPEC.md and is exactly the scheme the ticket asks about:

> "a metadata pair is stored in two blocks, with one block providing a backup during erase cycles in
> case power is lost"[^lfs-spec]
> "**Revision count (32-bits)** - Incremented every erase cycle. If both blocks contain valid
> commits, only the block with the most recent revision count should be used. Sequence comparison
> must be used to avoid issues with integer overflow."[^lfs-spec]
> "**CRC (32-bits)** - Detects corruption from power-loss or other write issues. Uses a CRC-32 with
> a polynomial of `0x04c11db7` initialized with `0xffffffff`."[^lfs-spec]

DESIGN.md is blunt about the requirement: "An embedded filesystem must be designed to recover from a
power loss during any write operation," and "Atomicity (a type of power-loss resilience) requires two
parts: redundancy and error detection."[^lfs-design] Upstream tests power loss directly — the repo
ships `tests/test_powerloss.toml` and an emulated block device `bd/lfs_emubd.c`, plus a file-backed
`bd/lfs_filebd.c` that makes host testing trivial.[^lfs-repo]

Caveats, all from primary sources:

- Wear levelling is **dynamic only**: "littlefs (currently) only provides dynamic wear leveling.
  This is a best effort solution. Wear is not distributed perfectly, but it is distributed among the
  free blocks."[^lfs-design] Comparable to NVS in kind; NVS's factor-of-126 append packing is
  arguably better for our tiny-record workload.
- **It is not a first-party ESP-IDF component.** The ESP-IDF v6.0 storage index lists FATFS, NVS,
  SPIFFS, SD/SDIO/MMC, partitions, VFS and wear levelling; LittleFS appears only as a pointer to the
  third-party component.[^storage-index] On the registry it is `joltwallet/littlefs`.[^lfs-registry]
- It is **absent from the ESP-IDF Linux/host support matrix**, which lists `nvs_flash`, `fatfs`,
  `spiffs` and `esp_partition` as simulated on Linux but has no row for littlefs.[^host-apps]
  (Upstream littlefs itself builds and tests on the host trivially — but the ESP wrapper is what we
  would actually be running on target, so the two would diverge.)
- Its partition needs subtype `0x83` (`ESP_PARTITION_SUBTYPE_DATA_LITTLEFS`), which the component
  itself `#define`s as a fallback "introduced in later patch versions of esp-idf".[^esp-lfs-src]
- "LittleFS filesystems can only grow, they cannot shrink."[^lfs-registry]

---

## Finding 3: SPIFFS — ruled out on Espressif's own words

> "When the chip experiences a power loss during a file system operation it could result in SPIFFS
> corruption. However the file system still might be recovered via `esp_spiffs_check`
> function."[^spiffs]

Also: "SPIFFS is able to reliably utilize only around 75% of assigned partition space" and garbage
collection "can take up to several seconds per write function call".[^spiffs] Rejected. It is only
mentioned here so the rejection is on record with a citation.

---

## Finding 4: FATFS, wear levelling, and the SD card

### FAT on SPI flash (FATFS + `wear_levelling`)

The `wear_levelling` component's *safety mode* is the only ESP-IDF storage layer with an explicit
power-loss statement of its own, and only about its own sector-remap step: in performance mode "if a
device is powered off for any reason, all 4096 bytes of data is lost", whereas in safety mode "If a
device is powered off, the data can be recovered as soon as the device boots up."[^wl] Note that
this is about the wear-levelling layer's internal copy, **not** about FAT metadata consistency.

The FATFS docs offer only `CONFIG_FATFS_IMMEDIATE_FSYNC` — "the FatFs will automatically call
`f_sync()` to flush recent file changes after each call of `write()`… This feature improves
file-consistency… at a price of decreased performance" — and warn that removing an open file without
`CONFIG_FATFS_FS_LOCK` is "undefined and may cause file system corruption".[^fatfs] Flushing is not
journalling. No ESP-IDF page claims FAT survives a power cut mid-update.

The clearest statement on this comes from littlefs's own comparison of ChaN's FatFs: "Due to
limitations of FAT it can't provide power-loss resilience".[^lfs-readme] That is a second party
describing FAT, not FAT's own documentation — but no primary FAT source claims otherwise, and the
absence of any atomicity or journalling mechanism in the format is not in dispute.

### The SD card on this board

M5Stack documents the PaperS3's microSD as an **SPI** interface: CS G47, SCK G39, MOSI G38, MISO
G40, on an ESP32-S3R8 with 8 MB PSRAM and a 16 MB external flash chip.[^papers3]

With no card present, `sdmmc_card_init()` fails during
`esp_vfs_fat_sdspi_mount()`/`esp_vfs_fat_sdmmc_mount()` and the mount aborts before any filesystem
work (`CHECK_EXECUTE_RESULT(err, "sdmmc_card_init failed")`).[^vfs-sdmmc] Detecting an absent card is
therefore easy and cheap. What is *not* documented anywhere I could find is what a specific SD card's
flash translation layer does when power is cut mid-program — that is vendor-internal and undocumented,
and I will not characterise it beyond saying it is unverifiable.

**Verdict: the SD card is optional and must never be on the save path.** It can be absent, it can be
removed at any moment by the user, it runs FAT, and keeping it powered costs battery in a device
whose budget (#12) is already tight. Legitimate uses: exporting the lifetime archive, staging sprite
assets during development, dumping care logs for balance analysis. The game must boot, hatch, live
and die with the slot empty.

---

## Finding 5: what NVS does *not* give us, and the atomicity scheme

### The gap

NVS gives us, per key, exactly what the ticket asks for. It does not give us:

1. **Multi-key atomicity.** Two `nvs_set_*` calls are two independent commits. Splitting pet state
   across keys reintroduces the torn-save bug one level up. → *Put all live pet state in one blob
   under one key.*
2. **A previous generation.** After a successful blob update NVS erases the old version
   immediately[^nvs-storage] and duplicate resolution prefers the newer value.[^nvs-pagemgr] There
   is nothing to fall back to. → *Keep our own A/B slots under two keys.*
3. **Protection against our own bugs.** A serialiser that writes a self-consistent but semantically
   wrong record passes every CRC NVS computes. → *Our own envelope, magic, schema version and CRC.*
4. **Schema versioning.** Not NVS's job. → *Ours.*
5. **A formal guarantee.** See the modality of the quote in Finding 1. → *Belt and braces.*

### Proposed partition table

16 MB of flash; there is no reason to be stingy.

```
# Name,      Type, SubType,  Offset,   Size,      Notes
nvs,         data, nvs,      0x9000,   0x6000,    system NVS: WiFi creds, PHY calib — NOT ours
phy_init,    data, phy,      0xF000,   0x1000,
factory,     app,  factory,  0x10000,  0x400000,  4 MB app
pet,         data, nvs,      0x410000, 0x10000,   64 KB — papergotchi save store
assets,      data, spiffs,   0x420000, 0x200000,  2 MB read-only sprite pack (optional, flashed image)
```

**Why a second NVS partition rather than sharing `nvs`.** WiFi provisioning (#19) and PHY
calibration write to the default `nvs` partition at their own cadence, contribute their own keys to
its mount time, and share its garbage collection. Isolating the pet gives it its own erase budget,
its own mount cost, and its own blast radius. Open it with `nvs_open_from_partition("pet", "pet",
…)`. 64 KB = 16 sectors, comfortably above the 0x3000 minimum.[^parttab]

### Keys within namespace `pet`

| Key | Type | Written | Contents |
| --- | --- | --- | --- |
| `slot.a`, `slot.b` | blob | alternating, every save | the full save envelope |
| `log.NNN` | blob | tail chunk only, on append | 32 care-log entries, self-describing |
| `grave.NNN` | blob | once, on death; never modified | one headstone |

`log.NNN` and `grave.NNN` indices live **inside the envelope**, not in separate keys — a separate
index key would reintroduce gap (1). Headstones are written once and never touched again, which is
the safest thing flash can do. Cap the on-device graveyard as a ring of the most recent N headstones
(64 ≈ 8 KB); the unbounded lifetime archive is already the cloud's job per the map's stated cloud
role, and #4 owns it.

### The envelope

```c
struct SaveEnvelope {           // little-endian; explicit field-by-field encode, never a struct memcpy
  uint32_t magic;               // 'PGCH'
  uint16_t schema_version;      // bumped on every incompatible change
  uint16_t payload_len;
  uint64_t sequence;            // monotonic; compare with wrapping arithmetic
  uint32_t payload_crc32;       // CRC-32, poly 0x04C11DB7, init 0xFFFFFFFF, over payload
  uint32_t header_crc32;        // CRC-32 over the preceding 20 bytes
  uint8_t  payload[payload_len];
};
```

Two CRCs, not one, so a truncated record is distinguishable from a corrupt one — that distinction
matters when deciding whether to blame the store or the serialiser. The wrapping sequence comparison
is the same trick littlefs uses for its revision counts: "Sequence comparison must be used to avoid
issues with integer overflow."[^lfs-spec]

`hatch_instant` is the first field of the payload and never moves between schema versions. It is the
one field whose loss is unrecoverable.

### Save algorithm

1. Append to the tail `log.NNN` chunk, or seal it and write a fresh one. *(Write-ahead: the thing
   the envelope will point at is committed before the envelope.)*
2. `target = (current_winner == "slot.a") ? "slot.b" : "slot.a"`.
3. Encode with `sequence = winner.sequence + 1`, compute both CRCs.
4. `nvs_set_blob(handle, target, buf, len)`, then `nvs_commit(handle)`.
5. On `ESP_OK`, flip `current_winner = target`. On any error, **do not flip** — the previous winner
   stands and the next save retries the same target.

**Invariant:** at every instant at least one of `slot.a`/`slot.b` holds a complete, CRC-valid
record. This holds because NVS never invalidates a blob's old version until the new version's
`BLOB_IDX` is committed[^nvs-storage] and we never touch both keys in one operation.

### Load and recovery algorithm

1. Read both slots. For each: length sane → `magic` matches → `header_crc32` matches →
   `schema_version` supported → `payload_crc32` matches. Any failure disqualifies the candidate;
   log which check failed and why (this is the diagnostic that tells us whether the invariant ever
   broke in the field).
2. Zero candidates → **there is no pet.** This is not the same as a dead pet. Consult the graveyard:
   if the most recent headstone's death instant is the last recorded event we are between pets;
   otherwise offer to hatch. **Never silently resurrect and never silently kill.**
3. One or two candidates → take the greater `sequence` (wrapping compare). The loser becomes the
   next write target.

### What the pet loses when the newest slot fails its checksum

Stated in game terms, as the ticket asks:

- **Elapsed years: nothing.** Age is `floor((now − hatch_instant) / 24h)` and the hatch instant is
  in every save. A one-generation rollback cannot change the pet's age by a single **year**.
- **Hearts: at most one, and only in a narrow window.** With saves fired on every **care event**,
  the exposed window is the duration of one `nvs_set_blob` — milliseconds — not the tick interval.
  If a brownout lands inside the save that follows a meal, the meal is undone: one Hunger **heart**
  and one unit of **weight**. The player can simply feed again.
- **Care log: at most one line, and it fails safe.** The log chunk is committed before the envelope.
  A crash between them leaves the line present and the state absent, which is the harmless
  direction: the line is dated, and the state that never committed never claimed the **care
  mistake**, so no duplicate is possible. A crash *during* the chunk rewrite leaves the previous
  chunk intact, losing at most the newest line.
- **Elapsed simulation: re-applied, not lost.** The core re-derives meters from
  `now − last_anchored_instant`, so replaying the rolled-back window produces the same answer —
  *provided* `step()` is pure. This is the requirement flagged in Finding 0 and it is the least
  obvious consequence of A/B rollback. If an illness roll or a **needless call** is drawn from
  unpersisted entropy, a rollback silently produces a different pet.

**This scheme is a good fit for ADR-0003.** Because care mistakes are stored as dated events rather
than as a monotonic counter, a rolled-back-and-replayed window cannot double-count them: the
counter that would have been incremented twice does not exist. Had we kept Gen1's hidden integer,
every rollback would have needed its own idempotence argument.

---

## Finding 6: flash wear — not the binding constraint

**Endurance figure.** The Winbond W25Q128JV datasheet — a 16 MB SPI NOR part of the class fitted to
boards like this — specifies "Min. 100K Program-Erase cycles per sector" and "More than 20-year data
retention" on page 1.[^w25q] GigaDevice's GD25Q128E, the other common candidate, is rated the same
(100,000 minimum cycles). **I could not verify which part M5Stack actually fitted to the PaperS3** —
their spec page says only "16MB" flash.[^papers3] The arithmetic below therefore assumes 100,000
minimum P/E cycles per 4 KB sector and should be re-run if the part turns out to be worse.

**Per-save flash cost.** A ~256-byte state payload becomes, in NVS's format: one `BLOB_IDX` entry +
one item-header entry + `ceil(256/32) = 8` data entries = **10 entries = 320 bytes** (1.25×
amplification). Allowing ~1.25× again for garbage-collection copying of live entries during page
recycling gives ≈400 bytes of flash traffic per save, against 4032 usable bytes per sector — call it
**0.1 sector-erases per save**, spread FIFO across the partition's sectors.[^nvs-pagemgr]

Over a five-year device life, on a 64 KB (16-sector) `pet` partition:

| Save cadence | Saves / 5 yr | Sector erases | Per sector | % of 100k budget |
| --- | --- | --- | --- | --- |
| Event-driven, ~24/day (recommended) | 44,000 | 4,400 | 275 | **0.3 %** |
| Every 5 min, 288/day (ticket premise) | 526,000 | 52,600 | 3,300 | **3.3 %** |
| Every 60 s, 1440/day (pathological) | 2,630,000 | 263,000 | 16,400 | **16 %** |

The care log adds ~100 tail-chunk rewrites per fortnight at ≤1 KB each — under 1 % of the above.
Graveyard headstones are written once per death (~26/year) and never modified.

**Conclusion: flash endurance is not the constraint.** Even the pathological cadence on the smallest
sensible partition leaves a large margin. Atomicity is the constraint, and the budget above is
exactly why we can afford to be generous with it — writing a save on every care event, rather than
batching, costs essentially nothing in wear and removes minutes of exposure.

---

## Finding 7: the same serialisation on the host

Three layers, and the top one never sees ESP-IDF:

1. **Encoder/decoder in the core.** Pure C++26 over `std::span<const std::byte>` /
   `std::span<std::byte>`, no ESP-IDF headers, no `nvs.h`. Explicit field-by-field encoding, never a
   `memcpy` of a struct — struct layout is not a stable format, and C++26's removal of several
   designated-initialiser idioms (noted in the map) is a reminder that compiler-dependent layout is
   a bad thing to commit to flash. Both hosts are little-endian (x86-64 and Xtensa LX7) but write
   the encoder as if they were not.
2. **A `SaveStore` port.** `load(slot) -> expected<bytes>` / `store(slot, bytes) -> expected<void>`.
   Target implementation wraps `nvs_get_blob`/`nvs_set_blob`; host implementations are an in-memory
   map (unit tests, fast-forward harness) and two files (desktop simulator, so a simulated pet
   survives restarting the simulator).
3. **Catch2 tests** for the envelope, winner selection, both-slots-corrupt, torn-payload, unknown
   schema version, and wrapping sequence comparison — all host-only, no ESP-IDF, milliseconds to
   run. Add a **golden-bytes test**: a hand-written hex literal that the encoder must reproduce
   exactly, so a schema change cannot pass silently.

**And then test the real thing on the host too.** ESP-IDF lists `nvs_flash` as *Simulation: Yes* and
`esp_partition` as *Mock: Yes, Simulation: Yes* on the Linux target.[^host-apps] The Linux
`esp_partition` implementation mmaps a temp file as emulated SPI NOR[^part-linux-h] and — critically
— exposes power-failure injection:

```c
/** @brief mode of fail after */
#define ESP_PARTITION_FAIL_AFTER_MODE_ERASE 0x01
#define ESP_PARTITION_FAIL_AFTER_MODE_WRITE 0x02
#define ESP_PARTITION_FAIL_AFTER_MODE_BOTH  0x03
...
void esp_partition_fail_after(size_t count, uint8_t mode);
```

> "Function initializes down counter emulating power off failure during write / erase operations.
> Once this counter reaches 0, actual as well as all subsequent write / erase operations
> fail"[^part-linux-h]

It also exposes `esp_partition_get_sector_erase_count()` per emulated sector[^part-linux-h] — which
means the wear table in Finding 6 can be *measured* rather than estimated, by running the
fast-forward harness and reading the counters.

This makes the central claim of this note directly testable in CI, on a laptop, with no hardware:
run the real NVS stack against the emulated partition, cut power at write *N* for *N* = 1…M, remount,
and assert the invariant — at least one slot loads, and the loaded record's sequence is either *S* or
*S*+1, never anything else. Espressif's own `test_nvs.cpp` uses exactly this pattern in its
`"test recovery from sudden poweroff"` monkey test;[^nvs-test] copy the shape of it.

---

## Recommendation

**Store live pet state, the care log and the graveyard in NVS, on a dedicated 64 KB `pet` partition,
as two alternating CRC-and-sequence-stamped slots under one namespace. Use no filesystem on the save
path. Treat the SD card as strictly optional and never write a save to it.**

Concretely:

1. Dedicated `data`/`nvs` partition `pet`, separate from the system `nvs`.
2. All live state in **one blob**, alternating between `slot.a` and `slot.b`, each carrying magic,
   schema version, monotonic sequence, and two CRC-32s.
3. Care log as immutable sealed chunks plus one rewritable tail chunk; graveyard as write-once
   headstones in a capped ring. Indices live inside the envelope.
4. Save on every **care event** and every state transition, plus a coarse heartbeat. Do not batch —
   Finding 6 shows we cannot afford *not* to.
5. Make `step()` pure and persist or derive all randomness, so rollback-and-replay is deterministic.
6. No LittleFS, no SPIFFS, no FAT, no SD for game data. Sprite assets go in the app image or a
   read-only flashed partition.
7. Test power loss on the host with `esp_partition_fail_after()` in CI, as a blocking test.

### Honest trade-offs

**Where LittleFS would genuinely be better.** Its power-loss claims are explicit, unhedged, and
specified in a format document, where NVS's are hedged prose ("does try to recover"). Appending to a
file costs nothing, where appending to an NVS blob rewrites the whole blob. It scales to an unbounded
graveyard without a cap. Its upstream has a dedicated power-loss test suite and a file-backed block
device for host testing. **If the care log or the graveyard were to grow into the megabytes, or if
the design needed genuine file semantics, LittleFS would be the right answer and this
recommendation should be revisited.**

**Why NVS wins anyway, here.** It is first-party and versioned with ESP-IDF, rather than a
third-party managed component that `#define`s its own partition subtype as a fallback.[^esp-lfs-src]
It is in the Linux host-support matrix; littlefs's ESP wrapper is not.[^host-apps] Its power-loss
behaviour is fuzz-tested in Espressif's own CI with the same injection mechanism we would use.
[^nvs-test] Its append-only entry log is a better fit for ≤32 KB of small records than a filesystem's
block allocator, and it avoids mounting a filesystem on a device that cold-boots from deep sleep
every few minutes — a cost that lands directly on #12's budget.

**What we give up.** The A/B slot scheme costs 2× the live-state footprint (trivial: ~640 bytes) and
one extra generation of staleness in the worst case. Rewriting a log tail chunk on append is
wasteful compared to a file append, though Finding 6 shows the waste is under 1 % of the erase
budget. The graveyard is capped on-device and depends on #4's cloud archive for the full lifetime
record — if #4 concludes there is no cloud, the graveyard question reopens and LittleFS comes back
into play.

**What this does not decide.** The save *schema* — the actual fields — is still open (the map lists
it under "not yet specified"), as is migration between schema versions. This note fixes the envelope
and the versioning field; it does not fix what is inside.

### Glossary gaps for `/domain-modeling`

`CONTEXT.md` has no entry for **graveyard**, **headstone**, or **save** / **snapshot**, all of which
this note and the map both use. Worth resolving before the schema ticket, so the save's field names
and the glossary agree.

---

## What remains uncertain

Stated plainly, because an unverified guarantee restated as fact is the failure mode this ticket
exists to prevent.

1. **NVS's power-loss behaviour is not a guarantee.** Espressif's own wording is "does try to
   recover" / "should not result in loss of data".[^nvs-doc] It is tested, not warranted. The A/B
   slot scheme exists precisely because I am not willing to bet a fortnight-old pet on that
   sentence.
2. **Half-erased sectors are the residual risk.** The docs and source cover aborted *writes*
   thoroughly. A brownout during a flash *erase* leaves a sector in a physically indeterminate state
   (partially erased cells that read as 1 but retain little charge and may drift). NVS has a
   `Corrupted` page state[^nvs-doc] and the `Erasing` state resumes on next boot,[^nvs-doc] but
   **I did not verify how NVS behaves against a sector left half-erased**, as opposed to one whose
   erase simply did not start. This is the single most important unverified item in this note.
3. **The flash part on the PaperS3 is unidentified.** M5Stack publishes "16MB" and no part
   number.[^papers3] The 100,000-cycle figure is from the W25Q128JV datasheet[^w25q] and is typical
   for the class, but it is not a measurement of this board. Read the chip marking and confirm.
4. **Brownout behaviour is uninvestigated.** Whether the ESP32-S3's brownout detector fires early
   enough to abort a flash operation cleanly, and what the PMS150G-mediated power-down sequence
   [^papers3] leaves time for, was not researched. This governs how often the torn-write case even
   arises, and it belongs with the power budget (#12).
5. **The "final snapshot before shutdown" has an unknown time budget.** The map promises the device
   "buzzes, takes a final cloud snapshot, and shuts down cleanly" on flat battery. Whether there is
   enough headroom below the brownout threshold to complete a flash write, let alone a WiFi upload,
   is unverified.
6. **SD card power-loss behaviour is unknowable from primary sources.** FTL internals are vendor
   secrets. This note asserts only that FAT provides no power-loss resilience[^lfs-readme] and that
   the card may be absent[^vfs-sdmmc] — both verifiable — and makes no claim about card-level
   corruption rates.
7. **The real save cadence is unknown** because it depends on the purity of a core that does not
   exist yet. Finding 6 brackets it; it does not settle it.
8. **littlefs's `block_cycles` tuning and its mount cost at this partition size were not measured**,
   because the recommendation avoids littlefs. If #4 or a later ticket reopens the filesystem
   question, that measurement is the first thing to take.

---

## Sources

All URLs retrieved 2026-08-02. ESP-IDF documentation is the **v6.0 / esp32s3** build; ESP-IDF source
citations are against `espressif/esp-idf` `master`, which may drift from the v6.0 tag.

[^nvs-doc]: ESP-IDF v6.0, *Non-Volatile Storage Library* (Introduction, Keys and Values, Security/Tampering/Robustness, and the whole Internals section: Log of Key-Value Pairs, Pages and Entries, Structure of a Page, Entry and Entry State Bitmap, Structure of Entry, Read-only NVS). <https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/storage/nvs_flash.html>
[^nvs-page]: `components/nvs_flash/src/nvs_page.cpp` — `Page::writeEntry()` (data write then `alterEntryState(..., WRITTEN)`), `Page::writeEntryData()`, and the half-written-entry recovery loop in `Page::load()`. <https://github.com/espressif/esp-idf/blob/master/components/nvs_flash/src/nvs_page.cpp>
[^nvs-storage]: `components/nvs_flash/src/nvs_storage.cpp` — `Storage::writeMultiPageBlob()` (chunks first, `BLOB_IDX` last), `Storage::writeItem()` (`VerOffset` toggle; old version erased only after commit), `Storage::init()`, `eraseMismatchedBlobIndexes()`, `eraseOrphanDataBlobs()`, `populateBlobIndices()`. <https://github.com/espressif/esp-idf/blob/master/components/nvs_flash/src/nvs_storage.cpp>
[^nvs-pagemgr]: `components/nvs_flash/src/nvs_pagemanager.cpp` — `PageManager::load()` duplicate-item resolution after power loss, and `PageManager::activatePage()` (`mFreePageList.pop_front()` / `push_back()` FIFO). <https://github.com/espressif/esp-idf/blob/master/components/nvs_flash/src/nvs_pagemanager.cpp>
[^nvs-api]: `components/nvs_flash/src/nvs_api.cpp` — `nvs_commit()`: "no-op for now, to be used when intermediate cache is added". <https://github.com/espressif/esp-idf/blob/master/components/nvs_flash/src/nvs_api.cpp>
[^nvs-test]: `components/nvs_flash/host_test/nvs_host_test/main/test_nvs.cpp` — `TEST_CASE("test recovery from sudden poweroff", "[long][nvs][recovery][monkey][.]")` and the surrounding power-off recovery cases, all driven by `NVSPartitionTestHelper::fail_after(...)`. <https://github.com/espressif/esp-idf/blob/master/components/nvs_flash/host_test/nvs_host_test/main/test_nvs.cpp>
[^parttab]: ESP-IDF v6.0, *Partition Tables* — default `single_factory_no_ota` layout and "It is strongly recommended that you include an NVS partition of at least 0x3000 bytes in your project." <https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-guides/partition-tables.html>
[^storage-index]: ESP-IDF v6.0, *Storage API* index. <https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/storage/index.html>
[^spiffs]: ESP-IDF v6.0, *SPIFFS Filesystem*, Notes. <https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/storage/spiffs.html>
[^fatfs]: ESP-IDF v6.0, *FAT Filesystem Support* — `CONFIG_FATFS_IMMEDIATE_FSYNC`, `CONFIG_FATFS_FS_LOCK`. <https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/storage/fatfs.html>
[^wl]: ESP-IDF v6.0, *Wear Levelling API* — performance vs safety mode. <https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32s3/api-reference/storage/wear-levelling.html>
[^vfs-sdmmc]: `components/fatfs/vfs/vfs_fat_sdmmc.c` — `sdmmc_card_init()` failure path in the mount sequence. <https://github.com/espressif/esp-idf/blob/master/components/fatfs/vfs/vfs_fat_sdmmc.c>
[^host-apps]: ESP-IDF, *Running ESP-IDF Applications on Host* — "Component Linux/Mock Support Overview" table (`nvs_flash`: Simulation Yes; `esp_partition`: Mock Yes / Simulation Yes; `fatfs`, `spiffs`: Simulation Yes; no littlefs row), and `idf.py --preview set-target linux`. <https://github.com/espressif/esp-idf/blob/master/docs/en/api-guides/host-apps.rst>
[^part-linux-h]: `components/esp_partition/include/esp_private/partition_linux.h` — `ESP_PARTITION_EMULATED_SECTOR_SIZE`, `ESP_PARTITION_FAIL_AFTER_MODE_*`, `esp_partition_fail_after()`, `esp_partition_get_sector_erase_count()`, and the mmap'd-temp-file emulation description. Implementation: `components/esp_partition/partition_linux.c`. <https://github.com/espressif/esp-idf/blob/master/components/esp_partition/include/esp_private/partition_linux.h>
[^lfs-readme]: littlefs upstream README — power-loss resilience, dynamic wear levelling, bounded RAM/ROM, POSIX-operation atomicity, and the FatFs comparison. <https://github.com/littlefs-project/littlefs/blob/master/README.md>
[^lfs-design]: littlefs upstream DESIGN.md — "The problem", "Metadata pairs", "Wear leveling". <https://github.com/littlefs-project/littlefs/blob/master/DESIGN.md>
[^lfs-spec]: littlefs upstream SPEC.md — "Directories / Metadata pairs", revision count and CRC field definitions. <https://github.com/littlefs-project/littlefs/blob/master/SPEC.md>
[^lfs-repo]: littlefs upstream repository layout — `tests/test_powerloss.toml`, `bd/lfs_emubd.c`, `bd/lfs_filebd.c`. <https://github.com/littlefs-project/littlefs>
[^lfs-registry]: ESP Component Registry, `joltwallet/littlefs` v1.22.3. <https://components.espressif.com/components/joltwallet/littlefs>
[^esp-lfs-src]: `esp_littlefs/src/esp_littlefs.c` — `#define ESP_PARTITION_SUBTYPE_DATA_LITTLEFS 0x83` with the comment "ESP_PARTITION_SUBTYPE_DATA_LITTLEFS was introduced in later patch versions of esp-idf", and the `esp_partition_find_first` lookups. <https://github.com/joltwallet/esp_littlefs/blob/master/src/esp_littlefs.c>
[^papers3]: M5Stack, *PaperS3* product documentation — ESP32-S3R8, 8 MB PSRAM, 16 MB flash, microSD over SPI (CS G47 / SCK G39 / MOSI G38 / MISO G40), BM8563 RTC, 1800 mAh battery, LGS4056H charger, PMS150G power control. <https://docs.m5stack.com/en/core/PaperS3>
[^w25q]: Winbond W25Q128JV datasheet, Features (page 1): "Min. 100K Program-Erase cycles per sector", "More than 20-year data retention". <https://cdn.sparkfun.com/assets/5/b/2/a/6/W25Q128JV_Datasheet.pdf>
