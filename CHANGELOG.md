# Changelog

## 1.1.0

- Distribution moved from GitHub Packages to Posto's public Maven repository at
  `https://postoconnect.com/sdk/maven`: no GitHub account or token is needed
  to install. Same coordinate, `com.posto:posto-android`.
- The sources jar is no longer published; the AAR is the compiled library
  only. Installation and usage documentation lives at
  <https://github.com/postoconnect/posto-android-sdk>.
- No API changes.

## 1.0.0

- Initial release, published to GitHub Packages as `com.posto:posto-android`.
- `PostoIntegrityClient`: Play Integrity Standard verdict bound to a
  non-exportable Android Keystore P-256 key; automatic key registration on the
  first ad request; signed impression and click callbacks without a second
  Google request.
- `PostoTestClient`: emulator lane against `POST /v1/ads/request_non_attest`
  using the App's Developer test key. Fixed, non-billable creative only.
- Deterministic PII filter over the latest three complete conversation rounds.
- Endpoint, header, and enum constants generated from the canonical
  `openapi/openapi.json` in the postoconnect monorepo
  (`scripts/sync_openapi_contract.py`).
