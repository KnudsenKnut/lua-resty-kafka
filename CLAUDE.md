# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

lua-resty-kafka is a Kafka client driver for OpenResty/ngx_lua, using the cosocket API for 100% nonblocking behavior. It implements the Kafka protocol in pure Lua and is designed to run within nginx worker processes.

**Requirements:**
- ngx_lua 0.9.3+ or OpenResty 1.4.3.7+ with LuaJIT (`--with-luajit`)
- For SSL connections: ngx_lua 0.9.11+ or OpenResty 1.7.4.1+
- For SCRAM-SHA-256/512 SASL: lua-resty-jit-uuid and lua-resty-openssl

## Development Commands

### Running Tests
```bash
# Run all tests (requires OpenResty and test::nginx::socket::lua)
make test

# Run specific test file
prove -I ../test-nginx/lib -r t/producer.t
```

**Test Environment Variables:**
Tests expect these environment variables (see t/producer.t):
- `TEST_NGINX_KAFKA_HOST`: Kafka broker host (default: 127.0.0.1)
- `TEST_NGINX_KAFKA_PORT`: Kafka broker port (default: 9092)
- `TEST_NGINX_KAFKA_SSL_PORT`: SSL port (default: 9093)
- `TEST_NGINX_KAFKA_SASL_PORT`: SASL port (default: 9094)
- `TEST_NGINX_KAFKA_SASL_USER/PWD`: SASL credentials (default: admin/admin-secret)

Tests use the Test::Nginx::Socket::Lua framework. Each test is embedded in nginx config blocks with Lua code.

### Installation
```bash
make install PREFIX=/usr/local
```

## Architecture Overview

### Core Module Hierarchy

1. **High-level APIs (User-facing)**
   - `resty.kafka.producer`: Sync/async producer with buffering
   - `resty.kafka.basic-consumer`: Simple consumer for fetching messages
   - `resty.kafka.client`: Metadata management and broker discovery

2. **Protocol Layer**
   - `resty.kafka.request`: Kafka request encoding (binary serialization)
   - `resty.kafka.response`: Kafka response decoding (binary deserialization)
   - `resty.kafka.protocol.common`: Common protocol utilities
   - `resty.kafka.protocol.consumer`: Consumer-specific protocol (ListOffsets, Fetch API)
   - `resty.kafka.protocol.record`: Record/message format handling

3. **Transport & Connection Layer**
   - `resty.kafka.broker`: Socket management, connection pooling, SASL authentication
   - `resty.kafka.sasl`: SASL mechanism encoding
   - `resty.kafka.scramsha`: SCRAM-SHA implementation

4. **Buffering & Data Management**
   - `resty.kafka.sendbuffer`: Per-topic-partition buffering for async producer
   - `resty.kafka.ringbuffer`: Circular buffer for async producer queue
   - `resty.kafka.errors`: Error code mappings from Kafka protocol

### Key Design Patterns

**Producer Architecture:**
- **Sync mode**: Sends immediately, returns offset on success
- **Async mode**:
  - Uses `ringbuffer` (circular queue) to buffer messages across all topics
  - Uses `sendbuffer` to organize messages by topic-partition before sending
  - Background timer (`ngx.timer.every`) flushes buffer based on `flush_time` or `batch_num`
  - Singleton per cluster_name within nginx worker (stored in `cluster_inited` table)
  - Buffering limits: `batch_num` (messages per batch), `batch_size` (bytes), `max_buffering` (total queue size)
  - Optional semaphore-based blocking when buffer is full (`wait_on_buffer_full`)

**Client Architecture:**
- Maintains metadata cache (brokers and topic partitions)
- Optional auto-refresh via `refresh_interval` using ngx.timer
- Supports API version negotiation via `choose_api_version`
- Handles broker discovery and leader selection

**Broker Architecture:**
- Socket pooling via ngx.socket.tcp with keepalive
- SASL authentication flow: handshake → authenticate
- Supports PLAIN, SCRAM-SHA-256, SCRAM-SHA-512 mechanisms
- SSL/TLS support with optional verification
- Custom DNS resolver function support

**Binary Protocol Handling:**
- Uses FFI (LuaJIT) for efficient int64 operations
- Manual bit manipulation for encoding (int8/16/32/64, strings, arrays)
- CRC32 for message integrity and partition selection
- Supports multiple Kafka API versions (v0, v1, v2, v3)

### Error Handling

Errors fall into three categories:
1. **Network errors**: Connection failures, timeouts (check connectivity)
2. **Metadata errors**: Missing topics/partitions, stale metadata (check broker/config)
3. **Kafka errors**: Error codes from broker responses (uppercase names like `OFFSET_OUT_OF_RANGE`)
   - See `lib/resty/kafka/errors.lua` for error code mappings
   - See Kafka protocol docs for error semantics

### API Version Compatibility

The library supports multiple Kafka versions through `api_version` configuration:
- Kafka 0.8.x: api_version = 0
- Kafka 0.9.x: api_version = 0 or 1
- Kafka 0.10.0.0+: api_version = 0, 1, or 2
- Kafka 1.0+: api_version = 3-8 supported
  - v3: Transactional/idempotent producer support (requires transactional_id)
  - v4: Same as v3, compatible with new message format
  - v5-v7: Added log_start_offset field in responses
  - v8: Enhanced error reporting with record-level errors
- Default: API_VERSION_V1 for most requests

Use `client:choose_api_version()` to automatically select compatible versions based on broker capabilities.

### Partitioning

Default partitioner (producer.lua:47-52):
```lua
function default_partitioner(key, num, correlation_id)
    local id = key and crc32(key) or correlation_id
    return id % num  -- partitions are 0-indexed
end
```
- If key provided: deterministic CRC32-based routing
- If no key: round-robin via auto-incrementing correlation_id

### Important Implementation Details

- **Correlation ID**: Auto-incrementing request ID (wraps at 2^30) for matching requests/responses
- **Worker Isolation**: Async producers are per-worker singletons (not shared across workers)
- **Timer Lifecycle**: Background timers are cancelled when `ngx.worker.exiting()` is true
- **Reusable Objects**: Message queues are reused up to MAX_REUSE (10000) times to reduce GC pressure
- **Batch Flushing**: Async producer flushes on batch_num threshold OR flush_time timeout (whichever comes first)
- **Transactional Producers**: API v3+ requires `transactional_id` field in requests for idempotent/transactional messaging
- **API Version Negotiation**: Client queries broker capabilities via ApiVersions API, then uses `choose_api_version()` to select highest mutually supported version

## Codebase Conventions

- Copyright headers reference Dejiang Zhu (doujiang24)
- Use `local ok, new_tab = pcall(require, "table.new")` pattern for table.new fallback
- FFI is used extensively for int64 operations (`ffi.cdef`, `*1LL` casting)
- Error returns follow `nil, err_string` pattern
- Configuration tables are passed through multiple layers (client_config, producer_config)
- Tests use nginx.conf-embedded Lua with Test::Nginx::Socket::Lua assertions
