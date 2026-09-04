# appstoreconnect-swift-sdk Upgrade — `tag-20260903` → `master`

**Merged:** 2026-09-03
**Previous base:** `tag-20260903` @ `5e23487d` (2026-03-07, PR #335)
**New base:** `AvdLee/appstoreconnect-swift-sdk` master @ `55fceaba` (2026-08-22)
**Upstream commits:** 21 (PRs #336, #337, #338, #345, #346, #347, #348)
**Spec version:** App Store Connect OpenAPI **4.2 → 4.4.1**
**Primary consumers:** `ai/tool/localtools/ToolAppStoreConnect.swift`, `views/tool/ToolAppStoreConnectSetting.swift`

Almost all of the churn is regenerated code: **543 files, +14,270 / -1,750**, of which 56 new entities and 47 new path types. The three things that actually matter to us are (1) the winback-offer price patch finally landing as real generated code instead of a rejected text patch, (2) `fields[appInfos]=kidsAgeBand` and the deprecated `ageRatingDeclaration` relationship being stripped, and (3) `Nomination*Request` relationship `data` arrays becoming non-optional — a **source-breaking** signature change, though one we do not call.

Verified: `swift build` of the package succeeds on macOS (warnings only, all `SubscriptionAvailability` deprecations). Neither of our two consumer files references any removed symbol.

---

## Spec version bumps

| Commit | Date | Spec |
|--------|------|------|
| `86367405` | 2026-03-30 | 4.2 → **4.3** |
| `9882bb2f` | 2026-06-09 | 4.3 → **4.4** |
| `cb4d3958` | 2026-07-15 | 4.4 → **4.4.1** |

---

## Highlights

### Winback offer prices are now generatable (PR #338)

`WinBackOfferPriceInlineCreate` used to generate as nothing but `type` + `id` — Apple's spec omits its `relationships`, and the repo's fix for that was a text patch against the spec JSON that **did not apply**, leaving a committed `Sources/OpenAPI/app_store_connect_api.json.rej` behind.

The patch was reimplemented inside the Swift generator (`SpecPatcher.ensureWinBackOfferPriceInlineCreateFields`), so the struct now generates with:

```swift
public struct Relationships: Codable {
    public var territory: Territory                              // required, type: territories
    public var subscriptionPricePoint: SubscriptionPricePoint?   // type: subscriptionPricePoints
}
```

`.rej` is deleted and dropped from the `Package.swift` `exclude:` list.

**Relevance to us:** `ToolAppStoreConnect.createWinbackOffer` currently posts `prices: .init(data: [])` because there was no way to express a price inline. That limitation is gone — the tool can now attach territory/price-point pairs at creation time if we want winback offers to be usable without a follow-up trip to App Store Connect.

`WinBackOfferCreateRequest.Attributes` also gains an optional `targetSubscriptionPlanType: SubscriptionPlanType?` (additive; our labeled-argument call site is unaffected).

### `kidsAgeBand` removed from `fields[appInfos]` (PR #347)

Apple's spec still lists `kidsAgeBand` as a valid `fields[appInfos]` value, but the live API rejects it with *"'kidsAgeBand' is not a valid field name"*. A new patch rule (`removeKidsAgeBandFromAppInfosFields`, `zeroMatchesIsOK: true`) walks every path/operation/parameter and filters the value out of the `fields[appInfos]` enum. The deprecated `AppInfo.kidsAgeBand` attribute and `fields[ageRatingDeclarations]` are deliberately left alone.

Effect: `FieldsAppInfos.kidsAgeBand` no longer exists on `APIEndpoint.v1.apps.get` / `apps.id().get` parameters. We never referenced it.

### Deprecated age-rating surface deleted (spec 4.4.1)

- `AppStoreVersion.Relationships.ageRatingDeclaration` and its nested `AgeRatingDeclaration` type are gone.
- `Include.ageRatingDeclaration` and the whole `FieldsAgeRatingDeclarations` enum are dropped from the app-store-version endpoints (`fields[ageRatingDeclarations]` no longer encoded).
- Two path types deleted: `PathsV1AppStoreVersionsWithIDAgeRatingDeclaration.swift`, `PathsV1AppStoreVersionsWithIDRelationshipsAgeRatingDeclaration.swift`.

**Source-breaking for anyone reading age ratings off a version.** We don't — no hit for `ageRating` in either consumer file.

### Nomination relationship arrays are required (PR #348)

Apple marks `relatedApps` / `inAppEvents` / `supportedTerritories` `data` arrays as optional, so the generated encoder could emit `"inAppEvents": {}`, which the live API rejects. `SpecPatcher.ensureNominationRelationshipDataArraysAreRequired` now forces `required: ["data"]` on those three relationships in `NominationCreateRequest` and `NominationUpdateRequest`.

Generated result — **source-breaking**:

```swift
// before
public var data: [Datum]?
public init(data: [Datum]? = nil)
// after
public var data: [Datum]
public init(data: [Datum])
```

An empty array now encodes as `[]` instead of being omitted. Covered by the new `Tests/NominationRequestEncodingTests.swift`. We do not use the Nominations API.

### `PromotedPurchases.WithID.get` signature change

`get(fieldsPromotedPurchases:include:)` became `get(parameters: GetParameters?)`, with the parameter struct gaining `fieldsInAppPurchases` and `fieldsSubscriptions`. **Source-breaking** for callers of that specific getter. Our tool only calls `promotedPurchases.id(_:).patch(_:)` and the collection-level `apps.id(_:).promotedPurchases.get(parameters:)`, both unaffected.

### Generator changes (`Tools/OpenAPIGeneratorCore/SpecPatching.swift`)

Two patch rules retired, three added:

| Removed | Added |
|---------|-------|
| `appPricev2InlineCreate-fields` (FB22160685) | `winbackofferpriceinlinecreate-fields` |
| `territoryavailabilityinlinecreate-fields` (FB22160701) | `remove-appinfo-kidsageband-field` |
| | `nomination-relationship-data-arrays` |

`Tests/SpecPatchingTests` extended accordingly (+108 lines).

---

## New API surface (spec 4.3 / 4.4 / 4.4.1)

### Versioned in-app purchase and subscription metadata

The largest addition: IAP/subscription metadata now has an explicit *version* object, mirroring how App Store versions work, plus V2 localizations and images.

New entities: `InAppPurchaseVersion`, `SubscriptionVersion`, `SubscriptionGroupVersion` (+ create/response/linkage variants), `InAppPurchaseLocalizationV2`, `SubscriptionLocalizationV2`, `SubscriptionGroupLocalizationV2`, `InAppPurchaseImageV2`, `SubscriptionImageV2`.

New endpoints include:

- `/v1/inAppPurchaseVersions[/{id}]` + `/image`, `/images`, `/localizations`, and their relationship links
- `/v1/subscriptionVersions[/{id}]` + `/image`, `/images`, `/localizations`, relationship links
- `/v1/subscriptionGroupVersions[/{id}]` + `/localizations`
- `/v1/subscriptions/{id}/versions`, `/v1/subscriptionGroups/{id}/versions`, `/v2/inAppPurchases/{id}/versions`
- `/v2/inAppPurchaseLocalizations[/{id}]`, `/v2/inAppPurchaseImages[/{id}]`
- `/v2/subscriptionLocalizations[/{id}]`, `/v2/subscriptionImages[/{id}]`, `/v2/subscriptionGroupLocalizations[/{id}]`

### Subscription plan availability

`SubscriptionPlanAvailability` + `SubscriptionPlanType` replace the now-**deprecated** `SubscriptionAvailability`:

- `/v1/subscriptionPlanAvailabilities[/{id}]` + `/availableTerritories`
- `/v1/subscriptions/{id}/planAvailabilities`

Deprecated: `SubscriptionAvailability`, `SubscriptionAvailabilityCreateRequest`, `SubscriptionAvailabilityResponse`, `SubscriptionAvailabilityAvailableTerritoriesLinkagesResponse`, `SubscriptionSubscriptionAvailabilityLinkageResponse`. (These are the only deprecation warnings in the package build.)

### Pricing

- `/v1/subscriptionPricePoints/{id}/adjustedEqualizations` — new.
- `/v3/appPricePoints/{id}` gained fieldsets/includes (+114 lines).

### Game Center

The whole v1 achievement / leaderboard / leaderboard-set surface is now **deprecated** in favor of the `/v2/gameCenter*` endpoints, which gained substantial fieldset and relationship coverage in this range. New `GameCenterActivityVersionInlineCreate` and `GameCenterChallengeVersionInlineCreate`; `GameCenterActivityVersionRelease` / `GameCenterChallengeVersionRelease` and their achievement/leaderboard linkage requests are deprecated.

### Smaller additions on endpoints we already call

- `/v1/builds` gained `fields[betaGroups]`, `fields[buildBundles]`, `fields[buildUploads]`.
- `/v1/appStoreVersions/{id}` and `/v2/appStoreVersionExperiments/{id}` gained `fields[apps]`.
- `/v1/reviewSubmissions` gained additional fieldsets (+106 lines).
- Many `App` fieldsets gained `accessibilityUrl`, `streamlinedPurchasingEnabled`, `accessibilityDeclarations`, `appTags`, `customerReviewSummarizations`.

---

## Housekeeping

- **CI**: simulator destinations pinned to `OS=latest` instead of hardcoded `26.0/26.1/26.2` (`4030c6c2`); `vapor/swiftly-action` `v0.2.0 → v0.2.1` (`d642eda7` — its message reads "Bump to 0.2.1" but it only touches the action pin, not any SDK version; the repo carries no version file).
- **README** (PR #336): `metada` typo fixed, Twitter badge → x.com.
- **Package.swift**: `OpenAPI/app_store_connect_api.json.rej` removed from `exclude:`. Note the pre-existing benign warning — `exclude:` still lists `app_store_connect_api.json.orig`, which is not in the repo.

---

## Action items for this project

1. **Nothing is required.** The package builds and neither consumer file touches a removed symbol.
2. **Optional follow-up:** `createWinbackOffer` in `ToolAppStoreConnect.swift` can stop sending `prices: .init(data: [])` now that `WinBackOfferPriceInlineCreate.Relationships` exists. Doing so means adding `territory` + `subscriptionPricePoint` parameters to the tool schema and to `helper/toolmapping_test.json`.
3. **Watch for:** if we ever add nomination or age-rating actions to the tool, both APIs changed shape in this range.
