# Posto Android SDK

Posto ad delivery for Android apps distributed through Google Play. The SDK
binds every ad request to a Play Integrity Standard verdict and a
non-exportable Android Keystore key (impressions and clicks are signed with
that key alone), filters PII on-device, and keeps tracking tokens private.

This repository holds the installation and usage documentation, the
changelog and the license. The library ships as a compiled AAR from Posto's
Maven repository; its source is not distributed.

## Requirements

- Android 7.0+ (`minSdk 24`), `compileSdk 35`, Java 17 toolchain.
- Kotlin coroutines: `requestAd()`, `trackImpression()` and `clickURL()` are
  `suspend` functions.
- App distributed through Google Play, with a Cloud project linked to Play
  Integrity. Emulators use the isolated test lane below.

## Installation

`com.posto:posto-android` is published to Posto's public Maven repository.
No account, token or GitHub access is needed.

1. Add the repository next to `google()` and `mavenCentral()` in
   `settings.gradle.kts`:

   ```kotlin
   dependencyResolutionManagement {
       repositories {
           google()
           mavenCentral()
           maven {
               name = "Posto"
               url = uri("https://postoconnect.com/sdk/maven")
           }
       }
   }
   ```

2. Depend on the library in your app module's `build.gradle.kts`. Pin the
   exact version; releases are described in [CHANGELOG.md](CHANGELOG.md):

   ```kotlin
   dependencies {
       implementation("com.posto:posto-android:1.1.0")
   }
   ```

   The AAR declares its own dependencies (Play Integrity, kotlinx-coroutines)
   and ships its consumer ProGuard rules, so no extra R8 configuration is
   needed.

3. Register the app in the Posto Developer console, then open its **App
   profile** and complete **Production connection**: Play package name, every
   accepted Play signing-certificate SHA-256 digest, the minimum version code,
   and the app's Google Cloud project (service-account JSON key). That page
   also shows the app's **Platform ID**, placement IDs and **Developer test
   key**. Billable delivery starts once the app passes publisher review; until
   then production requests return `403 publisher_delivery_suspended`.

> **Before you ship:** Play Integrity does not produce an acceptable verdict on
> emulators or sideloaded builds, and the SDK fails closed there by design.
> Upload the build through **Play Internal App Sharing**, install it from that
> link on a physical device, and verify one request → impression → click
> against `https://simulation.postoconnect.com`. Simulation is a separate stack
> with its own accounts: sign up there, register the same package name and
> certificate digest, and use that console's Platform ID and placement IDs.

## Usage

This Android library combines Play Integrity Standard with one non-exportable
P-256 key created in Android Keystore. The first call automatically registers
the key under a fresh integrity verdict. Every ad request is then bound to both
the Google verdict and that installation key; impression and click callbacks
use the same key without making another Google request.

```kotlin
val posto = PostoIntegrityClient(
    context = applicationContext,
    platformId = "your_platform_id",
    cloudProjectNumber = 123456789L,   // Long, from the Google Cloud console
)

posto.prepare() // Optional warm-up.

val context = PostoAdContext(
    interaction = PostoInteraction(listOf(
        PostoMessage(
            PostoMessageRole.USER,
            "Find a quiet hotel in Chicago",
        ),
        PostoMessage(
            PostoMessageRole.ASSISTANT,
            "Here are several downtown options.",
        ),
        PostoMessage(
            PostoMessageRole.USER,
            "Which one is under $250?",
        ),
    )),
    user = PostoUser(
        ageRange = PostoAgeRange.AGE_25_34,
        location = PostoLocation(
            country = "US",
            region = "Illinois",
            city = "Chicago",
        ),
    ),
)

val ad = posto.requestAd(PostoAdRequest(
    placementId = "plc_native_card",
    format = PostoAdFormat.NATIVE_CARD,
    context = context,
))

if (ad != null) {
    sponsoredLabel.text = PostoAd.SPONSORED_LABEL   // fixed English "Sponsored"; localize it yourself
    headlineLabel.text = ad.headline
    bodyLabel.text = ad.body
    ad.trackImpression()
    val destination = ad.clickURL()
}
```

`requestAd()` selects the latest three complete conversation rounds, applies
the deterministic PII filter, and returns `PostoAd?`; `null` is a normal
no-fill. `PostoAd` exposes only render fields plus `trackImpression()` and
`clickURL()`. Raw tracking tokens and the initial advertiser destination stay
private.

The public request does not accept `commerce`, `question`, `answer`, input
images, interests, analytics packets, or arbitrary dictionaries. Optional
coarse demographics are used only for explicit audience eligibility and never
enter semantic embeddings.

Register the Play package name, all accepted SHA-256 signing-certificate
digests, and a minimum version code in the Posto Developer console, then
connect the app's Google Cloud project: enable the Play Integrity API in the
project linked to the app in Play Console, create a service account, and
paste its JSON key into the app's production connection. Posto verifies the
key on save; until it is connected, attestation and attested ad requests fail
fast with `409 play_integrity_not_connected` (the SDK does not retry). The SDK
does not read or send Android ID, IMEI, advertising ID, serial number, or any
other hardware identifier. Registration sends only the installation's public
key and the app and SDK versions; Posto uses the key as an opaque
per-installation security subject.

The production client does not contain a Posto secret. Devices without an
acceptable Play Integrity verdict and apps outside Google Play must use a
Developer backend HMAC relay for production delivery. Emulators can use the
isolated test lane below, but never fall back to unsigned billable delivery.

## Emulator integration test

Use `PostoTestClient` with the App's **Developer test key** when validating the
SDK in an Android Emulator:

```kotlin
val posto = PostoTestClient(
    platformId = "app_your_platform",
    testKey = "your_developer_test_key",
    baseURL = "https://simulation.postoconnect.com",
)

val ad = posto.requestAd(PostoAdRequest(
    placementId = "plc_native_card",
    format = PostoAdFormat.NATIVE_CARD,
    context = context,
))
ad?.trackImpression()
val destination = ad?.clickURL()
```

The client calls `POST /v1/ads/request_non_attest` and receives Posto's fixed
test creative. The request, response, PII filtering, impression, and click
contracts are the same as production, but it does not enter an auction, spend
an advertiser budget, create Developer earnings, or train the Campaign Agent.
Production apps must keep using `PostoIntegrityClient`; the test client is not
an unsigned fallback for live delivery.

## Support

Open an issue in this repository for integration questions. Quote the
`requestId` of the affected ad request where you have one.

## License

Copyright (c) 2026 Posto, Inc. All rights reserved. See [LICENSE](LICENSE).
