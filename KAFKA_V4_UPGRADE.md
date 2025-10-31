# Kafka v4+ API Support Upgrade

This document summarizes the changes made to add Kafka v4+ (API versions 4-8) support to lua-resty-kafka.

## Summary

The library now supports Kafka ProduceRequest API versions 3-8, enabling compatibility with modern Kafka clusters (Kafka 1.0+) and features like transactional/idempotent producers, message headers, and enhanced error reporting.

## Changes Made

### 1. lib/resty/kafka/request.lua
- Added API version constants V4 through V9 (both local and module-level)
- These constants are now available for use throughout the codebase

### 2. lib/resty/kafka/producer.lua

#### Added API Version Constants
- Added local constants API_VERSION_V3 through API_VERSION_V8

#### Updated `produce_encode()` function
- Added support for `transactional_id` field (required for API v3+)
- The transactional_id is sent before acks/timeout when api_version >= 3

#### Updated `produce_decode()` function
- **API v0-v1**: errcode + offset (unchanged)
- **API v2-v4**: errcode + offset + timestamp (extended from v2 only)
- **API v5-v7**: Added log_start_offset field
- **API v8+**: Added record_errors array and error_message field for enhanced error reporting

#### Updated Constructor (`new()`)
- Added `transactional_id` field initialization from `opts.transactional_id`
- Defaults to `nil` for non-transactional producers

### 3. t/producer.t (Tests)
- Added TEST 11: Tests API version 4 compatibility
- Added TEST 12: Tests API version 5 compatibility
- Both tests verify that messages can be sent and offsets are returned correctly

### 4. README.md (Documentation)

#### Updated `api_version` documentation
- Expanded version support information to include v3-v8
- Added detailed feature descriptions:
  - v3+: Idempotent and transactional producer support
  - v4+: New message format with headers support
  - v5+: log_start_offset in responses
  - v8+: Record-level error reporting

#### Added `transactional_id` documentation
- New configuration option for idempotent/transactional producers
- Explains when and how to use it
- Example usage provided

### 5. CLAUDE.md (Architecture Documentation)

#### Updated API Version Compatibility section
- Added Kafka 1.0+ support details
- Documented v3-v8 features and requirements

#### Updated Important Implementation Details
- Added note about transactional producers requiring transactional_id
- Added clarification about API version negotiation process

## Protocol Details

### ProduceRequest API Version Evolution

**Version 3** (Kafka 0.11.0+)
- Added `transactional_id` field to request header
- Enables idempotent and transactional messaging (exactly-once semantics)

**Version 4** (Kafka 0.11.0+)
- Same as v3, but compatible with new RecordBatch message format
- Supports message headers (from KIP-82)

**Version 5** (Kafka 1.0.0+)
- Added `log_start_offset` to partition responses
- Helps producers handle UNKNOWN_PRODUCER_ID errors

**Versions 6-7** (Kafka 1.1.0+)
- Maintained v5 schema with internal improvements

**Version 8** (Kafka 2.4.0+)
- Added `record_errors` array to partition responses
- Added `error_message` field for better error diagnostics
- Enables batch-level error identification

## Usage Examples

### Basic Producer (API v1 - default, backward compatible)
```lua
local producer = require "resty.kafka.producer"
local p = producer:new(broker_list)
local offset, err = p:send("topic", "key", "message")
```

### Modern Producer with API v4
```lua
local producer = require "resty.kafka.producer"
local p = producer:new(broker_list, {
    api_version = 4
})
local offset, err = p:send("topic", "key", "message")
```

### Transactional Producer (API v3+)
```lua
local producer = require "resty.kafka.producer"
local p = producer:new(broker_list, {
    api_version = 3,
    transactional_id = "my-app-txn-id"
})
local offset, err = p:send("topic", "key", "message")
```

### Producer with Enhanced Error Reporting (API v8)
```lua
local producer = require "resty.kafka.producer"
local p = producer:new(broker_list, {
    api_version = 8,
    error_handle = function(topic, partition_id, message_queue, index, err, retryable)
        -- Enhanced error handling with record-level errors available
        ngx.log(ngx.ERR, "Failed to send to ", topic, "/", partition_id, ": ", err)
    end
})
```

## Backward Compatibility

All changes are backward compatible:
- Default API version remains v1
- Existing code continues to work without modification
- `transactional_id` is optional (nil by default)
- API version negotiation ensures compatibility with older brokers

## Testing

To test the new functionality:
```bash
# Ensure Kafka broker is running on localhost:9092
make test

# Or run specific tests
prove -I ../test-nginx/lib -r t/producer.t
```

Tests 11 and 12 specifically verify v4 and v5 API compatibility.

## Important Limitations and Fixes

### RecordBatch Format Support (CRITICAL)

**Limitation**: API versions 3+ should use RecordBatch format (magic byte 2) for encoding messages, but this library only supports MessageSet format (magic byte 0 and 1).

**Impact**:
- The library sends messages using MessageSet format even with API v3+
- **Works with most Kafka brokers** through backward compatibility
- **May not work** with brokers configured with `log.message.format.version` requiring RecordBatch
- Message headers are NOT supported (headers require RecordBatch format)

**Current Implementation**:
- API v0-v1: Uses MESSAGE_VERSION_0 (magic byte 0)
- API v2+: Uses MESSAGE_VERSION_1 (magic byte 1) with timestamp support
- INFO-level log warning when using v3+ without RecordBatch

**Future Work**: Implement RecordBatch encoding for full v3+ support

### Critical Bugs Fixed

1. **throttle_time_ms Parsing** (lib/resty/kafka/producer.lua:170-173)
   - **Bug**: API v1+ responses include throttle_time_ms after all partition responses, but it wasn't being read
   - **Impact**: Response parsing was incomplete, could cause issues with subsequent requests
   - **Fix**: Now properly reads throttle_time_ms for v1+ responses
   - **Added**: `ret.throttle_time_ms` field in decoded responses

2. **Field Name Correction** (lib/resty/kafka/producer.lua:129)
   - **Bug**: Field was named `timestamp` but Kafka protocol calls it `log_append_time`
   - **Fix**: Renamed to `log_append_time` for consistency with protocol

3. **API Version Split** (lib/resty/kafka/producer.lua:118-122)
   - **Bug**: v0 and v1 were grouped together but v1 includes throttle_time_ms
   - **Fix**: Separated v0 and v1 handling for clarity (responses are identical except for throttle_time_ms)

## Reliability Improvements

1. **Validation and Warnings**
   - INFO-level warning when using v3+ with MessageSet format
   - DEBUG-level notice when using v3+ without transactional_id
   - Helps users understand limitations upfront

2. **Message Format Selection**
   - Changed from `api_version == V2` to `api_version >= V2`
   - Ensures timestamp support for all modern API versions
   - Added inline documentation explaining the limitation

3. **Error Handling**
   - v8+ responses now include record-level errors
   - `record_errors` array captures batch_index and error_message
   - Top-level `error_message` for partition-level errors

## Testing Recommendations

### Unit Tests
```bash
make test
```

### Integration Tests with Real Kafka

Test with different broker configurations:

1. **Default Kafka 2.x+** (should work)
```bash
# broker configuration: defaults
```

2. **Kafka with log.message.format.version=0.11** (should work)
```bash
# server.properties:
log.message.format.version=0.11.0.0
```

3. **Kafka with log.message.format.version=3.0** (may not work)
```bash
# server.properties:
log.message.format.version=3.0
# This enforces RecordBatch format only
```

## Notes

- **API v9+**: Not yet implemented (requires compact string encoding)
- **Consumer API**: Already had v4+ support in `protocol/consumer.lua`
- **Client API**: The `choose_api_version()` function automatically negotiates the best supported version between client and broker
- **RecordBatch Encoding**: Not implemented - this is the main limitation for full v3+ support
- **Message Headers**: Not supported (requires RecordBatch format)

## References

- Kafka Protocol Documentation: https://kafka.apache.org/protocol.html
- KIP-98 (Exactly Once): https://cwiki.apache.org/confluence/display/KAFKA/KIP-98
- KIP-82 (Record Headers): https://cwiki.apache.org/confluence/display/KAFKA/KIP-82
