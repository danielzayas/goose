# Goose Project Test Coverage Analysis

## Summary

This document provides an analysis of test coverage for the Goose project, specifically focusing on:
1. Local test coverage and whether API calls to language model providers are mocked
2. Cross-service test coverage that makes real API calls to language model providers

---

## 1. Local Test Coverage - Mocking Strategy

### Answer: API calls are primarily mocked, but there are also tests that make real API calls

### Files Used to Determine This:

1. **`crates/goose/src/providers/testprovider.rs`**
   - Contains `TestProvider` which implements a record/replay pattern
   - Can wrap any provider to record interactions to a file
   - Can replay recorded interactions without making real API calls
   - Used extensively in scenario tests

2. **`crates/goose/tests/test_support.rs`**
   - Contains `ConfigurableMockScheduler` for mocking scheduler behavior
   - Contains helper functions for creating mock providers in tests

3. **`crates/goose/tests/agent.rs`**
   - Lines 590-610: Uses `MockProvider` struct that implements the `Provider` trait
   - Returns hardcoded responses without making real API calls
   - Example: `MockProvider` returns "Task completed." without network requests

4. **`crates/goose/tests/private_tests.rs`**
   - Uses mock schedulers (`ConfigurableMockScheduler`) for testing
   - No real API calls to language model providers in these tests

5. **`crates/goose/src/providers/factory.rs`**
   - Lines 184-197: Contains `MockTestProvider` for testing factory functionality

6. **Various provider implementation files:**
   - Multiple files contain `MockProvider` structs for unit testing
   - Examples found in:
     - `crates/goose/src/providers/lead_worker.rs` (lines 464-509)
     - `crates/goose/src/permission/permission_judge.rs` (lines 281-321)
     - `crates/goose/src/context_mgmt/auto_compact.rs` (lines 210-243)
     - `crates/goose/src/context_mgmt/summarize.rs` (lines 76-120)
     - `crates/goose-server/src/routes/reply.rs` (lines 545-564)

### Mocking Patterns Found:

1. **MockProvider Pattern**: Simple structs that implement the `Provider` trait and return hardcoded responses
2. **TestProvider Pattern**: Record/replay mechanism that can either:
   - Record real API calls to a file (when recording)
   - Replay recorded responses from a file (when replaying)
3. **WireMock**: Used for mocking HTTP servers in tests (found in OAuth and GCP auth tests)

---

## 2. Local Test Coverage - Real API Calls

### Tests That Make Real API Calls:

**File: `crates/goose/tests/providers.rs`**

This file contains integration tests that make **REAL API calls** to language model providers. These tests:

- Check for required environment variables (API keys) and skip if not present
- Make actual network requests to providers like:
  - OpenAI (`test_openai_provider`)
  - Anthropic (`test_anthropic_provider`)
  - Azure (`test_azure_provider`)
  - Bedrock (`test_bedrock_provider`)
  - Databricks (`test_databricks_provider`)
  - Google (`test_google_provider`)
  - Groq (`test_groq_provider`)
  - OpenRouter (`test_openrouter_provider`)
  - Ollama (`test_ollama_provider`)
  - Snowflake (`test_snowflake_provider`)
  - Xai (`test_xai_provider`)
  - LiteLLM (`test_litellm_provider`)
  - SageMaker TGI (`test_sagemaker_tgi_provider`)

- Test scenarios include:
  - Basic text responses
  - Tool usage
  - Context length exceeded errors
  - Image content support

**Key Evidence:**
- Lines 94-116: `test_basic_response()` calls `provider.complete()` which makes real API calls
- Lines 118-203: `test_tool_usage()` makes real API calls with tool definitions
- Lines 450-625: Individual test functions for each provider that require API keys

---

## 3. Cross-Service Test Coverage - Real API Calls

### Answer: Yes, there are tests that make real API calls to language model providers

### Files That Make Real API Calls:

#### 1. **`crates/goose/tests/pricing_integration_test.rs`**

**Makes REAL API calls to OpenRouter API**

- **Network Request**: `GET https://openrouter.ai/api/v1/models`
- **Test Functions**:
  - `test_pricing_cache_performance()` - Fetches pricing for multiple models
  - `test_pricing_refresh()` - Tests refreshing pricing data
  - `test_model_not_in_openrouter()` - Tests handling of non-existent models
  - `test_concurrent_access()` - Tests concurrent pricing fetches

**Evidence:**
- Line 279: `client.get("https://openrouter.ai/api/v1/models")`
- Uses `reqwest::Client` for HTTP requests
- Makes actual network calls and asserts on real responses
- Tests models like: `claude-3.5-sonnet`, `gpt-4o`, `gpt-4o-mini`, `gemini-flash-1.5`

**Implementation File**: `crates/goose/src/providers/pricing.rs`
- Line 279: `client.get("https://openrouter.ai/api/v1/models")`

#### 2. **`crates/goose/tests/mcp_integration_test.rs`**

**Can make REAL API calls (in record mode)**

- Uses `TestProvider` with record/replay pattern
- When `GOOSE_RECORD_MCP` environment variable is set, makes real API calls
- When replaying (default), uses recorded responses
- Tests MCP (Model Context Protocol) server integrations

**Evidence:**
- Lines 103-108: Checks for `GOOSE_RECORD_MCP` to determine record vs playback mode
- Lines 134-148: In record mode, sets up real commands and environment variables
- Tests real MCP servers like:
  - `@modelcontextprotocol/server-everything`
  - `github-mcp-server`
  - `mcp-server-fetch`
  - `goose-server` MCP developer tools

#### 3. **`crates/goose-cli/src/scenario_tests/scenario_runner.rs`**

**Can make REAL API calls (when recording)**

- Uses `TestProvider` for record/replay
- When recording file doesn't exist, makes real API calls to providers
- When replay file exists, uses recorded responses
- Tests full agent scenarios with real providers

**Evidence:**
- Lines 160-191: Determines replay vs record mode based on file existence
- Line 183: Creates real provider: `create(&factory_name, ModelConfig::new(&config.model_name)?)?`
- Line 185: Wraps in `TestProvider::new_recording()` to record real API calls
- Lines 173-179: Prevents recording in CI (GitHub Actions)

**Test Scenarios:**
- Tests complete agent workflows with real language model providers
- Records interactions for later replay
- Validates agent behavior across different providers

---

## Summary Table

| Test Type | File | Makes Real API Calls? | Provider Type | Notes |
|-----------|------|----------------------|---------------|-------|
| Unit Tests | `agent.rs`, `private_tests.rs` | ❌ No | Mock providers | Uses hardcoded responses |
| Provider Integration | `providers.rs` | ✅ Yes | All major providers | Requires API keys, skips if missing |
| Pricing Integration | `pricing_integration_test.rs` | ✅ Yes | OpenRouter API | Always makes real calls |
| MCP Integration | `mcp_integration_test.rs` | ✅ Yes (record mode) | MCP servers | Record/replay pattern |
| Scenario Tests | `scenario_runner.rs` | ✅ Yes (record mode) | All providers | Record/replay pattern |

---

## Key Findings

1. **Mocking Strategy**: The project uses multiple mocking strategies:
   - Simple `MockProvider` structs for unit tests
   - `TestProvider` with record/replay for integration tests
   - `WireMock` for HTTP server mocking

2. **Real API Calls in Local Tests**: 
   - `providers.rs` contains comprehensive integration tests that make real API calls
   - These tests require API keys and are skipped if credentials are missing

3. **Cross-Service Test Coverage**:
   - **`pricing_integration_test.rs`**: Always makes real API calls to OpenRouter
   - **`mcp_integration_test.rs`**: Makes real calls in record mode
   - **`scenario_runner.rs`**: Makes real calls when recording scenarios

4. **Record/Replay Pattern**: 
   - Used extensively to avoid making real API calls in CI
   - Allows developers to record real interactions locally
   - Replays recorded responses in automated test runs

---

## Files Referenced

### Core Test Infrastructure:
- `crates/goose/src/providers/testprovider.rs` - Record/replay provider
- `crates/goose/tests/test_support.rs` - Mock scheduler and test utilities

### Tests with Real API Calls:
- `crates/goose/tests/providers.rs` - Provider integration tests
- `crates/goose/tests/pricing_integration_test.rs` - OpenRouter API tests
- `crates/goose/tests/mcp_integration_test.rs` - MCP integration tests
- `crates/goose-cli/src/scenario_tests/scenario_runner.rs` - Scenario tests

### Implementation Files:
- `crates/goose/src/providers/pricing.rs` - Pricing API client

### Tests with Mocked Providers:
- `crates/goose/tests/agent.rs` - Agent tests with mocks
- `crates/goose/tests/private_tests.rs` - Scheduler tests with mocks
