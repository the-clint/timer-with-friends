# Research: React Native / Expo feasibility for a shared synchronized timer

Charting research for the wayfinder map. Date: **2026-09-30**. Version-sensitive.

Question: is React Native (via Expo) a proper fit for a small app that needs *both* push notifications and tasteful AdMob banners, and what are the traps?

## Verdict

Yes — arguably the lowest-friction route in 2026 for an app needing push + AdMob in one codebase: `expo-notifications` and `react-native-google-mobile-ads` are both first-class Expo config-plugin citizens.

## The load-bearing architectural finding

**You cannot run a live ticking clock in background JS.** iOS suspends background JS; Android Doze suspends background tasks; neither Expo, Apple, nor Google offers an SLA on background delivery; pushes are best-effort at-least-once. So the app must be architected as:

1. **Server-authoritative absolute end timestamp** (UTC) — clients render `endTime − now` locally, so all devices agree without continuous traffic.
2. **One OS-scheduled local notification** at the end time (`Notifications.scheduleNotificationAsync`) — Android AlarmManager + `RECEIVE_BOOT_COMPLETED`, iOS `UNUserNotificationCenter`; fires even when the app is killed, locked, or the device rebooted, with **zero JS running at fire time**.
3. **Remote push** (Expo Push Service, free) used only for *state-change nudges* (started / paused / resumed / cancelled), never to drive the clock.
4. **Foreground resync** — on open/resume, re-read authoritative state and re-derive the countdown.

## Facts

- **Expo SDK 57** (Jun 2026; React Native 0.86, New Architecture default, `expo prebuild` default). https://expo.dev/sdk/57
- `expo-notifications` provides a push token *and* OS-level local scheduling that survives kill/lock/reboot. https://docs.expo.dev/versions/latest/sdk/notifications.md · https://docs.expo.dev/push-notifications/what-you-need-to-know.md
- **Expo Push Service is free**: 600 notifications/sec/project, 100 messages/request (~4 KB payload), receipts polled async, best-effort no SLA. It fans out to APNs + FCM HTTP v1 for you. https://docs.expo.dev/push-notifications/faq.md · https://docs.expo.dev/push-notifications/sending-notifications.md
- FCM **HTTP v1** is the only supported FCM path in 2026 (legacy removed 2024).
- **Android 13 exact-alarm permission** (`SCHEDULE_EXACT_ALARM`) is required for exact-time scheduling; the SDK sets `RECEIVE_BOOT_COMPLETED` automatically; Android 8+ requires notification channels. https://docs.expo.dev/versions/latest/sdk/notifications.md#configurable-properties
- **Interval-based triggers drift** across DST / timezone / reboot; schedule against a **wall-clock date** and resync. https://rorklab.net/en/articles/rork/rork-expo-local-notification-timezone-dst-resync-design
- **`react-native-google-mobile-ads`** (Invertase) is the standard AdMob library and is Expo-first (`npx expo install` + config plugin). v17 needs RN 0.86+ New Architecture (fine with SDK 57), min iOS 15.1 / Android API 24. **Dev build only — not Expo Go.** https://github.com/invertase/react-native-google-mobile-ads
- Ad formats suited to "tasteful": anchored adaptive banner, inline/native in lists. Avoid interstitials/rewarded in a utility app. https://docs.page/invertase/react-native-google-mobile-ads/ad-formats/banner
- **EEA/UK consent** (UMP / GDPR) is required for ad serving; both stores require privacy disclosures for push + ads. https://docs.page/invertase/react-native-google-mobile-ads/european-user-consent

## Costs (one-time / annual, outside the monthly hosting ceiling)

- Apple Developer Program: **$99/year** (mandatory annual). Google Play: **$25 one-time** (personal accounts).
- New Google Play *personal* accounts must run a **closed test with ≥12 testers for 14 days** before production access. https://support.google.com/googleplay/android-developer/answer/14151465
- Ad revenue at tens of users ≈ **$0/day** (low-CPM utility inventory) — ads here are a statement feature, not income. [INFERENCE: standard AdMob eCPM ranges]

## Top gotchas

1. **No background JS ticking** — server-authoritative end timestamp + one OS-scheduled end notification + foreground resync; push is best-effort, not a clock.
2. **Push and ads both require a dev/EAS build** (not Expo Go) with native credentials (FCM v1, APNs). Verify the open ads config-plugin bug ([invertase #903](https://github.com/invertase/react-native-google-mobile-ads/issues/903)) before committing.
3. **Store/monetization friction**: ads at this scale earn ≈ $0 while still carrying EEA consent + privacy-disclosure obligations, and new Play accounts face the 12-tester / 14-day closed-test gate.

## Uncertain / version-sensitive

- Expo SDK / RN / ads-library versions and the #903 plugin bug are volatile; re-verify before building.
- The per-app AdMob eCPM figure is an estimate, not a verified 2026 number.

## Sources

Expo: [SDK 57](https://expo.dev/sdk/57) · [notifications](https://docs.expo.dev/versions/latest/sdk/notifications.md) · [push: what you need to know](https://docs.expo.dev/push-notifications/what-you-need-to-know.md) · [push FAQ](https://docs.expo.dev/push-notifications/faq.md) · [sending notifications](https://docs.expo.dev/push-notifications/sending-notifications.md)
Ads: [react-native-google-mobile-ads](https://github.com/invertase/react-native-google-mobile-ads) · [banner formats](https://docs.page/invertase/react-native-google-mobile-ads/ad-formats/banner) · [EU consent](https://docs.page/invertase/react-native-google-mobile-ads/european-user-consent)
Store: [Play closed testing](https://support.google.com/googleplay/android-developer/answer/14151465) · DST drift note: [rorklab](https://rorklab.net/en/articles/rork/rork-expo-local-notification-timezone-dst-resync-design)
