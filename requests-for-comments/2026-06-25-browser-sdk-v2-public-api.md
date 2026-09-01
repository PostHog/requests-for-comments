# Request for comments: Browser SDK v2 public API

- **Status:** Proposed
- **Decision maker:** @Team Client Libraries
- **Owner:** @dustinbyrne
- **Last updated:** 2026-07-16

## Summary

Browser SDK v2 should be a clean public API boundary for the JavaScript browser SDK. The v2 API removes long-deprecated
aliases, standardizes public JavaScript SDK naming on camelCase, makes feature flags return one consistent result
object, and stops exposing internal implementation objects as public API.

Core event transport and persisted identity/session semantics stay stable. The explicitly listed `$browser_type` and
`$config_defaults` property changes are the only event-property changes. This RFC makes the public API smaller, more
consistent with newer SDKs, and easier to document and maintain.

## Goals

- Keep the public SDK API focused on app-facing methods.
- Remove deprecated v1 aliases instead of carrying them into the next major version, except where ignoring an old
  privacy or consent key could increase collection.
- Standardize public JavaScript APIs and all SDK-owned option keys on camelCase instead of continuing the current mixed
  snake_case / camelCase surface.
- Keep v2 feature flag reads on the root `posthog` object.
- Keep internal objects (`posthog.featureFlags`, child implementation objects, underscored `PostHog` internals, and
  legacy extension globals) out of both the documented public API and the public unmangled namespace; reserve only one
  versioned private bridge for lazy extension bundles.

## Scope

This RFC covers the browser SDK v2 public API and config migration. All SDK-owned top-level and nested option keys move
to camelCase from the first v2 release. Except for the explicit `$browser_type` addition and `$config_defaults` removal,
event names, event `$properties`, request body fields, query/body wire protocol fields, third-party option objects passed
through verbatim, and persisted identity/session semantics stay stable.

Final npm package naming is tracked separately. This RFC applies to the browser v2 API whether it ships as `posthog-js`,
`@posthog/browser`, or both during a transition.

Product tours remains a shared extension that runs the same implementation in v1 and v2. Removing product tours code
is not part of the v2 release and can be decided independently.

Moving shared implementation into `@posthog/core` is also not a v2 prerequisite. We should evaluate it case by case
against browser bundle size, the cost of maintaining v1, and the size of the shared API surface.

Observable behavior/config changes covered by this RFC:

- Drop `$bot_pageview` routing.
- Adopt the latest staged v1 behaviors as the v2 baseline.
- Remove the dated `defaults` mechanism; future behavior changes ship through explicit package and script-loader
  versions.

## Decisions

### Feature flags

v2 uses one synchronous root feature flag read API:

```ts
const result = posthog.getFeatureFlag('new-onboarding')
if (result?.enabled) {
    console.log(result.variant, result.payload)
}
```

- `posthog.getFeatureFlag('name')` returns `FeatureFlagResult | undefined`. It returns `undefined` when flags are not
  available yet.
- `posthog.getFeatureFlagPayload()`, `posthog.getFeatureFlagResult()`, `posthog.getAllFeatureFlags()`, and
  `posthog.isFeatureEnabled()` are removed; their read semantics are folded into `getFeatureFlag()`.
- `posthog.onFeatureFlags(callback)` remains the subscription API for initial availability and later updates, with the
  callback signature `(flags: FeatureFlagResult[], errorsLoading: boolean) => void`.
- `posthog.featureFlags` is private and absent from public docs, types, and the unmangled `posthog` namespace.
- `posthog.featureFlags.override()` is removed. The root `posthog.updateFlags(flags, payloads?, { merge })` API is the
  public way to inject or override flag values.

A `FeatureFlagResult` is an object even when `enabled` is `false`. Existing truthiness checks such as
`if (posthog.getFeatureFlag('key'))` must migrate to `if (posthog.getFeatureFlag('key')?.enabled)`; otherwise a loaded
disabled flag would take the truthy branch.

### Remote feature flags

- Replace browser `advanced_disable_feature_flags` with the positive `featureFlagEvaluation` boolean, which defaults to
  `true`.
- With `featureFlagEvaluation: false`, reads use bootstrapped or runtime-supplied values instead of remote flag
  evaluation.
- Remote config remains available when `featureFlagEvaluation` is disabled.
- Drop `advanced_disable_flags` entirely. Remove the broader “kill remote config and flags APIs” switch.
- Drop `advanced_disable_decide` with `advanced_disable_flags`.

### Default behavior

- Remove the `defaults` config key and `$config_defaults` event property in v2.
- The first v2 release uses the baseline listed in [v2 baseline defaults](#v2-baseline-defaults).
- Future default changes require an explicit package or script-loader version rather than a dated config switch.

### Async init and extension loading

- v2 init is async and module callers await it before using the returned instance.
- App-facing calls made through the snippet or root singleton while init is pending—including `capture()`, `identify()`,
  registration, `setConfig()`, and consent methods—are queued in invocation order. Synchronous reads return their
  documented not-ready value instead of being queued.
- After base initialization, the SDK replays the full pre-init queue in order while network transmission remains paused.
  It begins sending only after configuration, persistence, and consent operations in that queue have been applied, so a
  queued opt-out can suppress earlier queued captures.
- If init fails, queued calls are discarded and a warning is emitted; the SDK must not transmit a queued capture with
  partially initialized privacy settings.
- Extension initialization is deferred by default.
- Remove `__preview_deferred_init_extensions` / `__previewDeferredInitExtensions`.

### Bot pageviews

`$bot_pageview` routing was a preview experiment: when bot detection matched and `__preview_capture_bot_pageviews` was
enabled, the SDK sent bot `$pageview` events as a separate `$bot_pageview` event instead of dropping them. Non-pageview
bot events kept their original names.

v2 removes this split event taxonomy. The SDK has one bot policy:

- Drop bot events by default.
- Use `optOutUseragentFilter` to opt out of user-agent filtering and capture bot traffic with the original event names.
- Add `$browser_type: 'bot' | 'browser'` when user-agent filtering is opted out.

This keeps pageview analytics and event naming consistent: `$pageview` remains the pageview event, and bot-vs-browser is
represented as a property when bot traffic is intentionally captured.

Implementation changes:

- Remove `__preview_capture_bot_pageviews`.
- Remove `$bot_pageview` routing.

## Why camelCase

The browser SDK currently has a mixed public naming surface: snake_case methods and config keys sit next to newer
camelCase APIs. v2 is the right boundary to standardize on idiomatic JavaScript naming across the public API.

This standardization makes the SDK easier to teach, document, type, and compare with newer SDK APIs. It also removes
pressure to carry both spellings forward as permanent aliases.

The rename applies to root methods and every SDK-owned option object from the first v2 release. Wire payload fields,
event properties, customer-defined keys, and third-party option objects passed through without interpretation keep
their existing names.

## Public root API changes

### Removed or replaced methods/properties

| v1 API                                                                     | v2 API                                                                                                     | Notes                                                                                 |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `posthog.people.set(props)` / `posthog.people.set('key', value)`           | `posthog.setPersonProperties(props)` / `posthog.setPersonProperties({ key: value })`                       | Callback argument is removed.                                                         |
| `posthog.people.set_once(props)` / `posthog.people.set_once('key', value)` | `posthog.setPersonProperties(undefined, props)` / `posthog.setPersonProperties(undefined, { key: value })` | Preserves `$set_once` behavior through the second argument.                           |
| `posthog.decideEndpointWasHit`                                             | No replacement                                                                                             | The internal request status is not a useful public contract.                          |
| `posthog._calculate_event_properties(...)`                                 | No public replacement                                                                                      | Event-property calculation remains private.                                           |
| `posthog.getFeatureFlag(key)` returning a raw flag value                   | `posthog.getFeatureFlag(key)` returning `FeatureFlagResult \| undefined`                                   | Root feature flag API returns value, variant, payload, and metadata together.         |
| `posthog.getFeatureFlagPayload(key)`                                       | `posthog.getFeatureFlag(key)?.payload`                                                                     | Payload is part of `FeatureFlagResult`.                                               |
| `posthog.getFeatureFlagResult(key)`                                        | `posthog.getFeatureFlag(key)`                                                                              | Result semantics move to the main API.                                                |
| `posthog.getAllFeatureFlags()`                                             | No replacement                                                                                             | Read individual flags through `getFeatureFlag()`.                                     |
| `posthog.isFeatureEnabled(key)`                                            | `posthog.getFeatureFlag(key)?.enabled`                                                                     | The result object replaces the boolean helper.                                        |
| `posthog.renderSurvey(surveyId, selector)`                                 | `posthog.displaySurvey(surveyId, { displayType: DisplaySurveyType.Inline, selector })`                     | `displaySurvey` also supports popovers and richer options.                            |
| `posthog.canRenderSurvey(surveyId)` / `posthog.canRenderSurveyAsync(...)`  | `const result = await posthog.canRenderSurvey(surveyId); result.visible`                                   | The async behavior moves to the unsuffixed method and returns a `SurveyRenderReason`. |
| `posthog.webPerformance`                                                   | No replacement                                                                                             | Deprecated compatibility shim removed.                                                |
| `posthog.featureFlags`                                                     | Private                                                                                                    | Absent from public docs, types, and the unmangled root namespace.                     |

### Root method renames

| v1                              | v2                           |
| ------------------------------- | ---------------------------- |
| `get_distinct_id()`             | `getDistinctId()`            |
| `get_session_id()`              | `getSessionId()`             |
| `get_session_replay_url()`      | `getSessionReplayUrl()`      |
| `get_property()`                | `getProperty()`              |
| `set_config()`                  | `setConfig()`                |
| `register_once()`               | `registerOnce()`             |
| `register_for_session()`        | `registerForSession()`       |
| `unregister_for_session()`      | `unregisterForSession()`     |
| `opt_in_capturing()`            | `optInCapturing()`           |
| `opt_out_capturing()`           | `optOutCapturing()`          |
| `has_opted_in_capturing()`      | `hasOptedInCapturing()`      |
| `has_opted_out_capturing()`     | `hasOptedOutCapturing()`     |
| `get_explicit_consent_status()` | `getExplicitConsentStatus()` |
| `clear_opt_in_out_capturing()`  | `clearOptInOutCapturing()`   |
| `is_capturing()`                | `isCapturing()`              |

Related entrypoint renames:

| v1                    | v2                  |
| --------------------- | ------------------- |
| `init_from_snippet()` | `initFromSnippet()` |
| `init_as_module()`    | `initAsModule()`    |

## Child/internal API cleanup

Child implementation objects are private in v2. Public use cases must go through a supported root method.

| v1 reachable API                                             | v2 public contract                                                     |
| ------------------------------------------------------------ | ---------------------------------------------------------------------- |
| `posthog.featureFlags.*`                                     | Root `getFeatureFlag()`, `onFeatureFlags()`, and `updateFlags()` only. |
| `posthog.toolbar._loadEditor(params)`                        | `posthog.loadToolbar(params)`.                                         |
| `posthog.toolbar.maybeLoadEditor(...)`                       | No public replacement; conditional loading is private.                 |
| `posthog.sessionRecording.onRRwebEmit(rawEvent)`             | No public replacement.                                                 |
| `posthog.experiments._is_bot()`                              | No public replacement; use a private utility if needed.                |
| `URLTriggerMatching.onRemoteConfig(response)`                | No public replacement.                                                 |
| `LinkedFlagMatching.onRemoteConfig(response, onStarted)`     | No public replacement.                                                 |
| `EventTriggerMatching.onRemoteConfig(response)`              | No public replacement.                                                 |
| `posthog.persistence` / `posthog.sessionPersistence` methods | No public replacement.                                                 |

The implementation may still rename private methods to camelCase for consistency, but those names are not migration
APIs. If a child-object use case proves necessary after release, add a narrow root method rather than exposing the
implementation object.

## Top-level config changes

Top-level config keys use camelCase as part of the standardization effort. Nested SDK-owned option keys follow the same
rule and are cataloged separately below.

### Active config key renames

| v1                                             | v2                                       |
| ---------------------------------------------- | ---------------------------------------- |
| `api_host`                                     | `apiHost`                                |
| `flags_api_host`                               | `flagsApiHost`                           |
| `ui_host`                                      | `uiHost`                                 |
| `api_transport`                                | `apiTransport`                           |
| `token`                                        | `projectToken`                           |
| `cross_subdomain_cookie`                       | `crossSubdomainCookie`                   |
| `persistence_name`                             | `persistenceName`                        |
| `cookie_persisted_properties`                  | `cookiePersistedProperties`              |
| `save_referrer`                                | `saveReferrer`                           |
| `save_campaign_params`                         | `saveCampaignParams`                     |
| `custom_campaign_params`                       | `customCampaignParams`                   |
| `custom_blocked_useragents`                    | `customBlockedUseragents`                |
| `capture_pageview`                             | `capturePageview`                        |
| `capture_pageleave`                            | `capturePageleave`                       |
| `disable_capture_url_hashes`                   | `disableCaptureUrlHashes`                |
| `cookie_expiration`                            | `cookieExpiration`                       |
| `disable_session_recording`                    | `disableSessionRecording`                |
| `disable_persistence`                          | `disablePersistence`                     |
| `persistence_save_debounce_ms`                 | `persistenceSaveDebounceMs`              |
| `split_storage`                                | `splitStorage`                           |
| `detect_google_search_app`                     | `detectGoogleSearchApp`                  |
| `disable_surveys`                              | `disableSurveys`                         |
| `disable_surveys_automatic_display`            | `disableSurveysAutomaticDisplay`         |
| `disable_product_tours`                        | `disableProductTours`                    |
| `disable_conversations`                        | `disableConversations`                   |
| `identity_distinct_id`                         | `identityDistinctId`                     |
| `identity_hash`                                | `identityHash`                           |
| `disable_web_experiments`                      | `disableWebExperiments`                  |
| `disable_external_dependency_loading`          | `disableExternalDependencyLoading`       |
| `strict_script_versioning`                     | `strictScriptVersioning`                 |
| `asset_host`                                   | `assetHost`                              |
| `prepare_external_dependency_script`           | `prepareExternalDependencyScript`        |
| `prepare_external_dependency_stylesheet`       | `prepareExternalDependencyStylesheet`    |
| `external_scripts_inject_target`               | `externalScriptsInjectTarget`            |
| `enable_recording_console_log`                 | `enableRecordingConsoleLog`              |
| `secure_cookie`                                | `secureCookie`                           |
| `opt_out_capturing_by_default`                 | `optOutCapturingByDefault`               |
| `opt_out_capturing_persistence_type`           | `optOutCapturingPersistenceType`         |
| `opt_out_persistence_by_default`               | `optOutPersistenceByDefault`             |
| `opt_out_useragent_filter`                     | `optOutUseragentFilter`                  |
| `consent_persistence_name`                     | `consentPersistenceName`                 |
| `opt_in_site_apps`                             | `optInSiteApps`                          |
| `respect_dnt`                                  | `respectDnt`                             |
| `property_denylist`                            | `propertyDenylist`                       |
| `request_headers`                              | `requestHeaders`                         |
| `on_request_error`                             | `onRequestError`                         |
| `request_batching`                             | `requestBatching`                        |
| `properties_string_max_length`                 | `propertiesStringMaxLength`              |
| `session_recording`                            | `sessionRecording`                       |
| `error_tracking`                               | `errorTracking`                          |
| `session_idle_timeout_seconds`                 | `sessionIdleTimeoutSeconds`              |
| `mask_all_element_attributes`                  | `maskAllElementAttributes`               |
| `mask_all_text`                                | `maskAllText`                            |
| `mask_personal_data_properties`                | `maskPersonalDataProperties`             |
| `custom_personal_data_properties`              | `customPersonalDataProperties`           |
| `advanced_disable_feature_flags`               | `featureFlagEvaluation`                  |
| `advanced_disable_feature_flags_on_first_load` | `advancedDisableFeatureFlagsOnFirstLoad` |
| `evaluation_contexts`                          | `evaluationContexts`                     |
| `flag_keys`                                    | `flagKeys`                               |
| `advanced_disable_toolbar_metrics`             | `advancedDisableToolbarMetrics`          |
| `advanced_only_evaluate_survey_feature_flags`  | `advancedOnlyEvaluateSurveyFeatureFlags` |
| `advanced_enable_surveys`                      | `advancedEnableSurveys`                  |
| `feature_flag_request_timeout_ms`              | `featureFlagRequestTimeoutMs`            |
| `feature_flag_cache_ttl_ms`                    | `featureFlagCacheTtlMs`                  |
| `advanced_feature_flags_dedup_per_session`     | `advancedFeatureFlagsDedupPerSession`    |
| `surveys_request_timeout_ms`                   | `surveysRequestTimeoutMs`                |
| `remote_config_refresh_interval_ms`            | `remoteConfigRefreshIntervalMs`          |
| `get_device_id`                                | `getDeviceId`                            |
| `before_send`                                  | `beforeSend`                             |
| `get_current_url`                              | `getCurrentUrl`                          |
| `capture_performance`                          | `capturePerformance`                     |
| `disable_compression`                          | `disableCompression`                     |
| `capture_heatmaps`                             | `captureHeatmaps`                        |
| `capture_dead_clicks`                          | `captureDeadClicks`                      |
| `capture_exceptions`                           | `captureExceptions`                      |
| `disable_scroll_properties`                    | `disableScrollProperties`                |
| `scroll_root_selector`                         | `scrollRootSelector`                     |
| `person_profiles`                              | `personProfiles`                         |
| `rate_limiting`                                | `rateLimiting`                           |
| `fetch_options`                                | `fetchOptions`                           |
| `request_queue_config`                         | `requestQueueConfig`                     |
| `cookieless_mode`                              | `cookielessMode`                         |
| `internal_or_test_user_hostname`               | `internalOrTestUserHostname`             |
| `override_display_language`                    | `overrideDisplayLanguage`                |
| `tracing_headers`                              | `tracingHeaders`                         |
| `disable_beacon`                               | `disableBeacon`                          |

`advanced_disable_feature_flags: true` maps to `featureFlagEvaluation: false`; it is the only active-config rename
that inverts a boolean.

Top-level keys that intentionally remain unchanged: `name`, `autocapture`, `rageclick`, `persistence`, `loaded`,
`debug`, `upgrade`, `surveys`, `logs`, `bootstrap`, `segment`, and `integrations`.

Internal `__extensionClasses` remains private.

Wire fields that look like config remain snake_case in payloads: project `token`, flags request body
`evaluation_contexts` / `flag_keys`, and conversations payload/query `identity_distinct_id` / `identity_hash`.

### Nested option key renames

All SDK-owned nested config types use camelCase. This catalog covers the existing snake_case keys; nested keys already
using camelCase are unchanged.

| Option object                  | v1 keys                                                                                                    | v2 keys                                                                                            |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `autocapture`                  | `url_allowlist`, `url_ignorelist`, `dom_event_allowlist`, `element_allowlist`                              | `urlAllowlist`, `urlIgnorelist`, `domEventAllowlist`, `elementAllowlist`                           |
| `autocapture`                  | `css_selector_allowlist`, `css_selector_ignorelist`, `element_attribute_ignorelist`, `capture_copied_text` | `cssSelectorAllowlist`, `cssSelectorIgnorelist`, `elementAttributeIgnorelist`, `captureCopiedText` |
| `rageclick`                    | `css_selector_ignorelist`, `content_ignorelist`, `ignore_text_selection`                                   | `cssSelectorIgnorelist`, `contentIgnorelist`, `ignoreTextSelection`                                |
| `rageclick`                    | `threshold_px`, `click_count`, `timeout_ms`                                                                | `thresholdPx`, `clickCount`, `timeoutMs`                                                           |
| `capturePerformance`           | `network_timing`, `web_vitals`, `web_vitals_allowed_metrics`                                               | `networkTiming`, `webVitals`, `webVitalsAllowedMetrics`                                            |
| `capturePerformance`           | `web_vitals_delayed_flush_ms`, `web_vitals_attribution`                                                    | `webVitalsDelayedFlushMs`, `webVitalsAttribution`                                                  |
| `errorTracking`                | `exception_steps`                                                                                          | `exceptionSteps`                                                                                   |
| `errorTracking.exceptionSteps` | `max_bytes`                                                                                                | `maxBytes`                                                                                         |
| `captureExceptions`            | `capture_unhandled_errors`, `capture_unhandled_rejections`, `capture_console_errors`                       | `captureUnhandledErrors`, `captureUnhandledRejections`, `captureConsoleErrors`                     |
| `captureDeadClicks`            | `scroll_threshold_ms`, `selection_change_threshold_ms`, `mutation_threshold_ms`                            | `scrollThresholdMs`, `selectionChangeThresholdMs`, `mutationThresholdMs`                           |
| `captureDeadClicks`            | `capture_clicks_with_modifier_keys`, `css_selector_ignorelist`, `element_attribute_ignorelist`             | `captureClicksWithModifierKeys`, `cssSelectorIgnorelist`, `elementAttributeIgnorelist`             |
| `captureHeatmaps`              | `flush_interval_milliseconds`                                                                              | `flushIntervalMilliseconds`                                                                        |
| `sessionRecording`             | `full_snapshot_interval_millis`, `compress_events`, `session_idle_threshold_ms`                            | `fullSnapshotIntervalMillis`, `compressEvents`, `sessionIdleThresholdMs`                           |
| `requestQueueConfig`           | `flush_interval_ms`                                                                                        | `flushIntervalMs`                                                                                  |
| `FeatureFlagOptions`           | `send_event`                                                                                               | `sendEvent`                                                                                        |

Private `__*` knobs are not renamed into public options. The implementation checklist includes an audit of every public
option type so newly added or less common SDK-owned objects follow the same rule.

### Removed config aliases and dead keys

| Removed v1 key                                  | v2 replacement / behavior                   | Notes                                                                                                                 |
| ----------------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `sanitize_properties`                           | `beforeSend`                                | Compatibility preserves `$set_once`; direct `beforeSend` migrations must handle it explicitly.                        |
| `ip`                                            | None                                        | v1 warned this config key had no effect. Use project-level “Discard IP data” or transformations for IP handling.      |
| `defaults`                                      | None                                        | v2 uses its package/script-loader baseline directly.                                                                  |
| `on_xhr_error`                                  | `onRequestError`                            | Signature changes from XHR-focused callback to request response callback.                                             |
| `xhr_headers`                                   | `requestHeaders`                            | Alias removed.                                                                                                        |
| `process_person`                                | `personProfiles`                            | Alias removed.                                                                                                        |
| `advanced_disable_decide`                       | None                                        | Deprecated `/decide` kill switch removed.                                                                             |
| `advanced_disable_flags`                        | None                                        | Broad remote-config/flags kill switch removed.                                                                        |
| `cookie_name`                                   | `persistenceName`                           | Alias removed.                                                                                                        |
| `disable_cookie`                                | `disablePersistence`                        | Alias removed.                                                                                                        |
| `store_google`                                  | `saveCampaignParams`                        | Alias removed.                                                                                                        |
| `verbose`                                       | `debug`                                     | Alias removed.                                                                                                        |
| `property_blacklist`                            | `propertyDenylist`                          | v1 merged blacklist into denylist; v2 reads `propertyDenylist`.                                                       |
| `__preview_disable_beacon`                      | `disableBeacon`                             | Preview alias removed.                                                                                                |
| `__preview_external_dependency_versioned_paths` | `strictScriptVersioning` and/or `assetHost` | v1 accepted boolean/string and mapped to supported options.                                                           |
| `__preview_deferred_init_extensions`            | None                                        | Async v2 init defers extensions by default.                                                                           |
| `evaluation_environments`                       | `evaluationContexts`                        | Config-side feature flag evaluation context.                                                                          |
| `enable_heatmaps`                               | `captureHeatmaps`                           | Superseded.                                                                                                           |
| `opt_out_capturing_cookie_prefix`               | `consentPersistenceName`                    | `consentPersistenceName` is the full storage key; the token is no longer appended as a prefix.                        |
| `_onCapture`                                    | Split by use case                           | Use `posthog.on('eventCaptured', ...)` for observe-only behavior, or `beforeSend` for mutation/filtering before send. |
| `__add_tracing_headers`                         | `tracingHeaders`                            | Preview/internal alias removed.                                                                                       |
| `addTracingHeaders`                             | `tracingHeaders`                            | Alias removed.                                                                                                        |
| `api_method`                                    | None                                        | No reader.                                                                                                            |
| `inapp_protocol`                                | None                                        | No reader.                                                                                                            |
| `inapp_link_new_window`                         | None                                        | No reader.                                                                                                            |
| `__preview_flags_v2`                            | None                                        | Concluded experiment.                                                                                                 |
| `__preview_eager_load_replay`                   | None                                        | Concluded experiment.                                                                                                 |
| `__preview_lazy_load_replay`                    | None                                        | Concluded experiment.                                                                                                 |
| `__preview_disable_xhr_credentials`             | None                                        | Concluded experiment.                                                                                                 |
| `__preview_cookie_wins_on_conflict`             | None                                        | Cookie-wins merge behavior is the only behavior.                                                                      |
| `__preview_capture_bot_pageviews`               | None                                        | `$bot_pageview` routing removed.                                                                                      |

### Safety-critical v1 key compatibility

The clean v2 boundary must not cause the SDK to collect or persist more data because JavaScript silently accepted an
old key. Throughout v2, the SDK also reads the v1 spelling when the v2 spelling is absent, applies the same behavior,
and emits one migration warning per key and instance. If both spellings are present, the v2 key wins and a conflict
warning identifies the ignored v1 key.

This compatibility allowlist covers:

- Consent and persistence: `cross_subdomain_cookie`, `cookie_persisted_properties`, `disable_persistence`,
  `disable_cookie`, `secure_cookie`, every `opt_out_*` key, `consent_persistence_name`, `respect_dnt`, and
  `cookieless_mode`.
- Capture enablement and filtering: `capture_pageview`, `capture_pageleave`, `disable_session_recording`,
  `session_recording`, `error_tracking`, `capture_performance`, `capture_heatmaps`, `enable_heatmaps`,
  `capture_dead_clicks`, `capture_exceptions`, `disable_scroll_properties`, `opt_out_useragent_filter`,
  `person_profiles`, and `process_person`.
- Redaction and pre-send controls: `disable_capture_url_hashes`, `property_denylist`, `property_blacklist`,
  `before_send`, `sanitize_properties`, `mask_all_element_attributes`, `mask_all_text`,
  `mask_personal_data_properties`, and `custom_personal_data_properties`.
- Nested autocapture allowlists/ignorelists and attribute filters, nested exception-capture switches, and nested
  dead-click allowlists/ignorelists and attribute filters listed in the nested migration catalog.

Other removed aliases and dead keys are ignored after a warning; they are not silently translated.

## v2 baseline defaults

The v2 baseline includes the latest default behavior that had already been staged in v1:

| Config key                    | v2 baseline default                                                                                 | v1 undated default |
| ----------------------------- | --------------------------------------------------------------------------------------------------- | ------------------ |
| `capturePageview`             | `'history_change'`                                                                                  | `true`             |
| `rageclick`                   | `{ contentIgnorelist: DEFAULT_CONTENT_IGNORELIST_WITH_STEPPERS, ignoreTextSelection: true }`        | `true`             |
| `sessionRecording`            | `{ strictMinimumDuration: true, canvasCapture: { resolutionScale: 0.6 }, streamNetworkBody: true }` | `{}`               |
| `externalScriptsInjectTarget` | `'head'`                                                                                            | `'body'`           |
| `internalOrTestUserHostname`  | `/^(localhost\|127\.0\.0\.1)$/`                                                                     | `undefined`        |
| `persistenceSaveDebounceMs`   | `250`                                                                                               | `0`                |
| `splitStorage`                | `true`                                                                                              | `false`            |
| `detectGoogleSearchApp`       | `true`                                                                                              | `false`            |
| `disableCaptureUrlHashes`     | `true`                                                                                              | `false`            |

Users should use individual config overrides to preserve older behavior.

## Window globals and extension contract removals

For lazy-loaded extension bundles, `window.__PosthogExtensions__` is the supported contract. The toolbar loader still
uses direct `window.ph_load_toolbar`; legacy `window.ph_load_editor` is removed.

| Removed global                         | v2 contract                                                        |
| -------------------------------------- | ------------------------------------------------------------------ |
| `window.rrweb`                         | `window.__PosthogExtensions__.rrweb`                               |
| `window.rrwebConsoleRecord`            | `window.__PosthogExtensions__.rrwebPlugins.getRecordConsolePlugin` |
| `window.getRecordNetworkPlugin`        | `window.__PosthogExtensions__.rrwebPlugins.getRecordNetworkPlugin` |
| `window.posthogErrorWrappingFunctions` | `window.__PosthogExtensions__.errorWrappingFunctions`              |
| `window.postHogWebVitalsCallbacks`     | `window.__PosthogExtensions__.postHogWebVitalsCallbacks`           |
| `window.postHogTracingHeadersPatchFns` | `window.__PosthogExtensions__.tracingHeadersPatchFns`              |
| `window.extendPostHogWithSurveys`      | `window.__PosthogExtensions__.generateSurveys`                     |
| `window.ph_load_editor`                | `window.ph_load_toolbar`                                           |

Also remove `window.__PosthogExtensions__.loadWebVitalsCallbacks()`.

## Survey and remote-config type cleanup

| Type/member                             | v2 status | Replacement / notes                                          |
| --------------------------------------- | --------- | ------------------------------------------------------------ |
| `SurveyAppearance.descriptionTextColor` | Removed   | Deprecated and unused.                                       |
| `SurveyAppearance.inputBackgroundColor` | Removed   | Use `inputBackground`.                                       |
| `ActionStepType.tag_name`               | Removed   | Use `selector`.                                              |
| `SurveyConfig.autoSubmitIfComplete`     | Removed   | No longer used; prefilled skip-submit behavior is automatic. |
| `SurveyConfig.autoSubmitDelay`          | Removed   | No longer used.                                              |
| `RemoteConfig.editorParams`             | Removed   | Use `toolbarParams`.                                         |
| `RemoteConfig.toolbarVersion`           | Removed   | Obsolete toolbar metadata.                                   |

## Private-by-default `PostHog` internals

Anything not declared in the public v2 interface is private, absent from the unmangled `posthog` namespace, and free to
be renamed or mangled. A v1 member does not receive a stable non-underscored name merely because internal modules or
external code could reach it.

| v1 reachable member                                                | v2 status                                                                                   |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `__loaded`, `_init()`, `_onRemoteConfig()`, `_dom_loaded()`        | Private; no public replacement.                                                             |
| `_send_request()`, `_send_retriable_request()`, `_execute_array()` | Private; no public replacement.                                                             |
| `_addCaptureHook()`, `_internalEventEmitter`, `_is_bot()`          | Private; no public replacement.                                                             |
| `_shouldDisableFlags()`                                            | Removed with `advanced_disable_flags`.                                                      |
| `_isIdentified()`                                                  | `isIdentified()` is promoted to the public root API.                                        |
| `_overrideSDKInfo()`                                               | Wrapper SDKs use `overrideSDKInfo(posthog, ...)` from the explicit `./internal` entrypoint. |

The `./internal` entrypoint is the typed integration boundary for first-party wrappers. It is excluded from generated
customer references and may contain only contracts that must cross package boundaries. Lazy extension bundles use the
versioned `window.__PosthogExtensions__` bridge described above; this is the sole required global implementation
bridge and is also excluded from public types and customer documentation.

All other implementation state, including request queues, persistence objects, extension instances, capture hooks, and
initialization helpers, remains private.

## Snippet and queued-call migration

The snippet queue dispatches by method name. Queued calls must use v2 method names.

Examples:

```js
posthog.setConfig({ disableSessionRecording: true })
posthog.registerOnce({ plan: 'pro' })
posthog.getDistinctId()
```

Old queued calls such as `posthog.set_config(...)`, `posthog.register_once(...)`, or `posthog.get_distinct_id()` fail in
v2 because the queue processor dispatches by exact method name. Calls using valid v2 names remain queued while async
init is pending under the ordering and failure rules above.

## Runtime behavior for removed aliases

- Removed methods, child objects, and properties are absent at runtime.
- Unknown config keys in plain JavaScript produce a migration warning and are otherwise ignored.
- The safety-critical compatibility allowlist translates its v1 spellings throughout v2 when the v2 key is absent.
- Any additional runtime alias must be an explicit compatibility shim with a warning and removal plan, rather than
  accidental public API.

## Migration and documentation

- Generate and host separate v1 and v2 SDK references so existing v1 deep links remain valid and v2 exposes only the
  supported public interface. The reference build must validate links for both versions.
- Audit the official browser SDK docs and generated references for every child object, underscored member, and extension
  global made private by this RFC. Existing v1 documentation should point to the v2 root replacement where one exists.
- Generate and maintain a browser v1-to-v2 migration skill from the same API/config catalog used for references. The
  skill should cover package installation, camelCase rewrites, async init, feature-flag truthiness, safety-key warnings,
  and manual migrations that cannot be codemodded.
- Keep the reference generator, migration skill, TypeScript declarations, and runtime migration warnings derived from
  one machine-readable rename/removal catalog so they cannot drift independently.

## Implementation checklist

- [ ] Implement root `getFeatureFlag()` returning `FeatureFlagResult | undefined`; remove the other public read helpers.
- [ ] Update `onFeatureFlags()` to emit `FeatureFlagResult[]` and `errorsLoading` while retaining update subscriptions.
- [ ] Replace `advanced_disable_feature_flags` with `featureFlagEvaluation` and test the inverted boolean migration.
- [ ] Remove `defaults` and `$config_defaults`; apply and test the complete v2 baseline.
- [ ] Make async init queue supported root/snippet calls in order and safely discard them on initialization failure.
- [ ] Rename `token` to `projectToken` in config while preserving wire/event `token` fields.
- [ ] Rename every SDK-owned top-level and nested option key, backed by an exhaustive public-type audit.
- [ ] Implement and test the safety-critical v1 key allowlist, precedence, and warning behavior.
- [ ] Remove child implementation objects and private members from public types and the unmangled instance namespace.
- [ ] Add the narrow `./internal` wrapper entrypoint and migrate wrapper SDKs away from `_overrideSDKInfo()`.
- [ ] Keep product tours on the shared v1/v2 extension path.
- [ ] Generate versioned v1/v2 references, run link validation, and publish the v1-to-v2 migration skill.

## Feedback requested

@Team Client Libraries is the decision maker for this RFC. Feedback should focus on:

- Known customer or wrapper SDK usage of removed aliases or child objects.
- Whether `featureFlagEvaluation` is the clearest positive name for controlling remote evaluation.
- Whether the retained `onFeatureFlags(FeatureFlagResult[], errorsLoading)` subscription covers update use cases.
- Any debug/test flag override use cases beyond `posthog.updateFlags(flags, payloads?, { merge })`.
- Any migration risks around async init queueing, changed baseline behavior, bot pageviews,
  `sanitize_properties` / `$set_once`, privacy-key compatibility, or split storage.
