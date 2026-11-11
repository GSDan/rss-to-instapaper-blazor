# 05 - Testing Strategy & Core Unit Test Plan

## 1. Test Architecture & Standards
- **Test Framework:** xUnit 2.x
- **Assertion Library:** FluentAssertions
- **Mocking:** NSubstitute
- **Database Fixtures:** `Microsoft.Data.Sqlite` in-memory with shared connections
- **Coverage Goal:** >= 85% on domain services, parsing logic, retry algorithms, and encryption.

---

## 2. Core Unit Test Specifications

### Module A: RSS Parsing & Backfill Period Filtering
- **Class:** `RssFeedParserServiceTests`
- **Scenarios:**
  1. `ParseFeedAsync_WithBackfillFilter_FiltersItemsOlderThanCutoff`: Verifies that items published prior to the computed cutoff date are tagged as `SkippedBacklog`.
  2. `CalculateCutoffDate_ValidInputs_CalculatesAccurateTimestamp`: Tests parameter matrix (`Days`, `Weeks`, `Months`) against expected date boundaries.

### Module B: Content Extraction & Fallback Rules Engine
- **Class:** `ArticlePayloadResolverTests`
- **Scenarios:**
  1. `ResolvePayload_DirectContentPreferred_ReturnsDirectBodyWhenPresent`: Confirms `<content:encoded>` HTML is prioritized when present.
  2. `ResolvePayload_DirectContentPreferredButMissing_FallsBackToUrlWhenEnabled`: Confirms fallback to URL scrape mode when body is absent and fallback is allowed.
  3. `ResolvePayload_DirectContentMissingAndFallbackDisabled_ThrowsValidationException`: Verifies exception/failure behavior when fallback is disabled.

### Module C: Exponential Backoff & Retry Logic
- **Class:** `IngestionRetryPolicyTests`
- **Scenarios:**
  1. `CalculateNextRetryUtc_ComputesExponentialInterval`: Verifies retry delays (Attempt 1 = 15m, Attempt 2 = 60m, Attempt 3 = 240m).
  2. `HandleFailure_ExceedingMaxRetries_TransitionsArticleToPermanentFailedState`: Confirms that after 3 failed attempts, status transitions to `Failed`, nullifies `NextRetryUtc`, and logs error message.

### Module D: Secret Encryption & Protection
- **Class:** `TokenEncryptionServiceTests`
- **Scenarios:**
  1. `ProtectAndUnprotect_WithPersistedDataProtection_PreservesExactSecrets`: Validates round-trip encryption/decryption of OAuth tokens and secrets using `IDataProtector`.
