# FaceTime Audio preset

## Supported profile

The preset is limited to the independently verified layout below:

| Property | Value |
| --- | --- |
| ProductType | iPhone18,1 (iPhone 17 Pro) |
| iOS / build | 27.3 / 24C5037i |
| CacheUUID | 7F1DF278-18B7-4E5A-8425-09E8830DE9AF |
| CacheData length | 6480 bytes (0x1950) |

The owner of a China-mainland device with this profile confirmed that FaceTime Audio worked after applying the two flag changes. Other models and builds need separate layout analysis before adding a profile.

## Usage

Read MobileGestalt from the device, then open **Device → Presets → FaceTime Audio**:

- **Keep Current** discards the staged FaceTime Audio selection.
- **Enable** stages `green-tea = false`, `not-green-tea = true`.
- **Restore Restriction** stages `green-tea = true`, `not-green-tea = false`. This restores the restriction combination, rather than claiming to recover an unknown factory value.

Review Changes shows each modified flag and its before/after value. Apply to Device uses the existing backup and full-plist readback transaction. The successful-write message asks the user to restart the iPhone and test a call. Imported plists remain local documents.

## Verified cache layout

Offsets are relative to the decoded top-level `CacheData` blob, not the plist file or dylib file.

| Query | Obfuscated key | Cache index | Value offset | Validity offset |
| --- | --- | ---: | --- | --- |
| green-tea | iyfxmLogGVIaH7aEgqwcIA | 383 | 0xBF8 | 0x17FF |
| not-green-tea | 4snMZS8LJkSctKypt2m+xA | 506 | 0xFD0 | 0x187A |

The layout contains 720 eight-byte value slots followed by 720 one-byte validity markers: `720 * 8 + 720 = 6480`. The reader checks `CacheData[0x1680 + index] == 1`. For query type 11 it reads the first byte at `index * 8` and converts nonzero to true. Only the two value bytes are edited; validity markers and slot padding are preserved.

The profile checks model, build, CacheUUID, length, both validity bytes, canonical Boolean bytes and zero slot padding. It also rejects conflicting query entries in CacheExtra. Commit validates the profile again and rejects changes to unrelated cache bytes.

## Reproducible analysis anchors

Analyzed Xcode-extracted arm64e `libMobileGestalt.dylib`:

- Mach-O UUID: `288A263B-3BEF-3A0F-B9AC-D2FE21C4961E`.
- SHA256: `df85094b2da0c6544707de6e00695ac726fb3932b727c473861b3caae84f764a`.
- Named getters: `_MobileGestalt_get_greenTeaDeviceCapability` at unslid VA `0x1AFF1CCF0`, and `_MobileGestalt_get_notGreenTeaDeviceCapability` at `0x1AFF3E3E0`.
- Their CFStrings resolve to `green-tea` and `not-green-tea`; both getters call `_MGGetBoolAnswer`.
- Query hashes are Base64 of MD5 of `MGCopyAnswer` followed by the clear query name, without trailing `=`.
- The lookup at `0x1AFF1AC60` uses weights at `0x1AFF8CCFC` and a 64-byte descriptor table at `0x1E551F7A0`.
- `green-tea` resolves to table position 834, descriptor VA `0x1E552C820`; `not-green-tea` resolves to position 396, descriptor VA `0x1E5525AA0`.
- Descriptor `+0x3A` holds the signed cache index; `+0x3C` holds the query type. Both entries have type 11.
- Cache reads at `0x1AFF1AD50..0x1AFF1AE28` establish the validity and value offsets above.
- The loader at `0x1AFF1B2CC..0x1AFF1B310` copies the plist's CacheData bytes into that cache buffer. `0x1AFF1B9CC..0x1AFF1BA20` checks its 0x1950-byte length.
- The plist's CacheUUID matches the library's compiled layout UUID.

These are unslid virtual addresses. Xcode's extracted image retains shared-cache encoded pointers; the low 34 bits are runtime offsets, as defined in Apple's [fixup-chains.h](https://github.com/apple-oss-distributions/dyld/blob/main/include/mach-o/fixup-chains.h). Decode them against the shared-cache base before following descriptor or CFString pointers.

The matching FTServices function at `0x1C261C874` checks the equivalent of `venice && (!green-tea || carrierOverrideFTA)`. Its log identifies the three inputs as `supportsFT`, `carrierOverrideFTA`, and `GT`. This describes the regional capability gate, rather than the entire account-registration or calling flow.

## Validation

The document tests cover exact-byte edits, unchanged regions and unrelated fields, idempotence, restriction restoration, model/build/layout rejection, malformed slots, CacheExtra conflicts, commit-time revalidation, staged cache preservation when another preset is reverted, and cache-only write/backup/restore transactions.

```sh
swiftc -module-cache-path /tmp/gestaltx-module-cache \
  GestaltX/GestaltDocument.swift GestaltX/FaceTimeAudioCache.swift \
  GestaltX/Capabilities.swift tests/DocumentTests.swift \
  -o /tmp/gestaltx-document-tests
/tmp/gestaltx-document-tests
```

## Credit

The original enablement idea and the two target flags come from [KirbySD's V2EX post](https://www.v2ex.com/t/1246550). The offsets above were independently derived from the matching 24C5037i libraries; templates from other builds were not reused. No device export, Apple binary, signing material or device identifier is included in this contribution.
