# Local Testing Guide

This guide shows you how to test lua-resty-kafka locally before pushing to GitHub Actions.

## Quick Start with Docker

### 1. Start Kafka Locally

```bash
# Start Kafka with all required configurations (PLAINTEXT, SSL, SASL)
docker-compose up -d

# Wait for Kafka to be ready (about 30 seconds)
docker-compose logs -f kafka-setup

# You should see: "Topics created successfully!"
```

### 2. Run Tests

```bash
# Run all tests
make test

# Run specific test
prove -I ../test-nginx/lib -r t/producer.t

# Run with verbose output
prove -v -I ../test-nginx/lib -r t/producer.t
```

### 3. Stop Kafka

```bash
docker-compose down

# Or to clean up volumes:
docker-compose down -v
```

---

## What's Included

The Docker Compose setup provides:

- ✅ **Kafka 3.6.x** on port 9092 (PLAINTEXT)
- ✅ **SSL** on port 9093 (with self-signed cert)
- ✅ **SASL/PLAIN** on port 9094
- ✅ **SCRAM-SHA-256 and SCRAM-SHA-512** authentication
- ✅ **Pre-created topics** (test, test2, test3, test4, test5, test-consumer)
- ✅ **Legacy message format** (log.message.format.version=2.4)
- ✅ **ZooKeeper** (required for older Kafka APIs)

---

## Testing Different Scenarios

### Test API Version 1 (default)
```bash
# This is what most tests use
prove -I ../test-nginx/lib -r t/producer.t::1
```

### Test API Version 4
```bash
# Our new test
prove -I ../test-nginx/lib -r t/producer.t::11
```

### Test API Version 5
```bash
# Our new test with log_start_offset
prove -I ../test-nginx/lib -r t/producer.t::12
```

### Test SSL
```bash
prove -I ../test-nginx/lib -r t/producer.t::2
```

### Test SASL/PLAIN
```bash
prove -I ../test-nginx/lib -r t/producer.t::8
```

### Test SASL/SCRAM-SHA-256
```bash
prove -I ../test-nginx/lib -r t/producer.t::9
```

### Test SASL/SCRAM-SHA-512
```bash
prove -I ../test-nginx/lib -r t/producer.t::10
```

---

## Troubleshooting

### Kafka won't start

**Check logs:**
```bash
docker-compose logs kafka
```

**Common issues:**
- Port 9092 already in use → Change port in docker-compose.yml
- Not enough memory → Increase Docker memory limit

**Solution:**
```bash
docker-compose down -v
docker-compose up -d
```

### Tests fail with "connection refused"

**Check if Kafka is ready:**
```bash
docker-compose ps
# Should show kafka as "healthy"

# Test connection
docker-compose exec kafka kafka-broker-api-versions --bootstrap-server localhost:9092
```

**Wait longer:**
```bash
# Kafka takes 20-30 seconds to start
sleep 30
make test
```

### SSL tests fail

**Check SSL certificate:**
```bash
docker-compose exec kafka ls -la /etc/kafka/secrets/
# Should show kafka.keystore.jks
```

**Regenerate certificate:**
```bash
docker-compose down -v
rm -rf docker/kafka-ssl/
docker-compose up -d
```

### SASL tests fail

**Check SCRAM credentials:**
```bash
docker-compose exec kafka kafka-configs \
  --zookeeper zookeeper:2181 \
  --describe \
  --entity-type users \
  --entity-name admin
```

**Should show:**
```
SCRAM-SHA-256=[password=admin-secret]
SCRAM-SHA-512=[password=admin-secret]
```

### Topics not created

**Check topics:**
```bash
docker-compose exec kafka kafka-topics --list --bootstrap-server localhost:9092
```

**Recreate topics:**
```bash
docker-compose restart kafka-setup
```

---

## Advanced Testing

### Test with Different Kafka Versions

Edit `docker-compose.yml`:
```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0  # Kafka 3.6.x
    # OR
    image: confluentinc/cp-kafka:7.8.0  # Kafka 3.9.x
    # OR
    image: confluentinc/cp-kafka:8.0.0  # Kafka 4.0.x (if available)
```

Then:
```bash
docker-compose down
docker-compose up -d
```

### Test with RecordBatch Format Required

Edit `docker-compose.yml` to enforce RecordBatch:
```yaml
# Change:
KAFKA_LOG_MESSAGE_FORMAT_VERSION: '2.4'
# To:
KAFKA_LOG_MESSAGE_FORMAT_VERSION: '3.0'
```

This will cause tests to fail (expected) because we don't support RecordBatch encoding yet.

### Interactive Debugging

Start a shell in the Kafka container:
```bash
docker-compose exec kafka bash

# Inside container:
kafka-console-producer --bootstrap-server localhost:9092 --topic test
# Type messages and Ctrl+C to exit

kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
# See messages
```

---

## Performance Testing

### Measure throughput

```bash
# Producer performance
docker-compose exec kafka kafka-producer-perf-test \
  --topic test \
  --num-records 10000 \
  --record-size 100 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092

# Consumer performance
docker-compose exec kafka kafka-consumer-perf-test \
  --bootstrap-server localhost:9092 \
  --topic test \
  --messages 10000
```

---

## CI Simulation

To simulate exactly what GitHub Actions does:

```bash
# 1. Start fresh
docker-compose down -v

# 2. Start Kafka
docker-compose up -d

# 3. Wait for setup to complete
docker-compose logs -f kafka-setup
# Wait until you see "Topics created successfully!"

# 4. Run tests
make test

# 5. Check results
echo $?  # Should be 0 for success
```

---

## Comparing with GitHub Actions

### Local Setup vs CI

| Feature | Local (Docker) | GitHub Actions |
|---------|---------------|----------------|
| Kafka Version | 3.6.x (configurable) | 2.4.0 → needs update |
| Startup Time | ~30 seconds | ~10 seconds (too short!) |
| SSL | ✅ Self-signed cert | ✅ Self-signed cert |
| SASL | ✅ All mechanisms | ✅ All mechanisms |
| Topics | Auto-created | Manual creation |
| Health Checks | ✅ Yes | ❌ No (needs adding) |

### Why CI Fails But Local Works

1. **Startup time**: CI uses `sleep 5`, should be `sleep 15` minimum
2. **No health checks**: CI doesn't verify Kafka is ready
3. **Old Kafka version**: CI uses 2.4.0, we test with 3.6.x
4. **ZooKeeper timing**: CI doesn't wait for ZooKeeper to be ready

---

## Fixing CI Based on Local Tests

Once tests pass locally, update `.github/workflows/test.yaml`:

```yaml
# Add after Kafka start (line ~77)
sudo /usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties

# Change from:
sleep 5

# To:
sleep 15  # Give Kafka more time to start

# Add health check:
timeout 60 bash -c 'until /usr/local/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092 &> /dev/null; do sleep 2; done'
echo "Kafka is ready!"
```

---

## Next Steps

1. ✅ Test locally with `docker-compose up -d && make test`
2. ✅ Verify all tests pass
3. ✅ Fix any issues locally first
4. ✅ Update CI workflow with lessons learned
5. ✅ Push to GitHub and watch CI pass

---

## Quick Reference

```bash
# Start Kafka
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f kafka

# Run tests
make test

# Stop Kafka
docker-compose down

# Clean everything
docker-compose down -v
rm -rf docker/kafka-ssl/
```

---

## Questions?

**Q: Do I need OpenResty installed locally?**
A: Yes, you need OpenResty to run the tests. But Kafka runs in Docker, so that's easier!

**Q: Can I test without Docker?**
A: Yes, but you'd need to manually install Kafka, ZooKeeper, configure SSL, SASL, etc. Docker is much easier!

**Q: Will this work on Windows/Mac?**
A: Yes! Docker Compose works on all platforms. WSL2 recommended for Windows.

**Q: How do I test just the code changes without running tests?**
A: Start Kafka, then manually connect:
```lua
local producer = require "resty.kafka.producer"
local p = producer:new({{host="127.0.0.1", port=9092}}, {api_version=4})
local offset, err = p:send("test", nil, "hello")
print(offset, err)
```
