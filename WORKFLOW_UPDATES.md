# GitHub Actions Workflow Updates

## Summary

The current `.github/workflows/test.yaml` needs updates to work with Kafka v4+ changes and modern tooling.

---

## Required Changes

### 1. **Kafka Version** (CRITICAL)

**Current:**
```yaml
KAFKA_VER: 2.4.0    # Released Dec 2019 (5+ years old)
SCALA_VER: 2.11
```

**Updated:**
```yaml
KAFKA_VER: 3.6.1    # Latest stable Kafka 3.x
SCALA_VER: 2.13     # Kafka 3.x requires Scala 2.13
```

**Why:**
- Kafka 2.4.0 technically supports API v8, but is very outdated
- Kafka 3.x has better API version handling
- Tests broader compatibility
- Kafka 2.x is end-of-life

**Alternatives:**
- `KAFKA_VER: 2.8.2` - If you want to stay on 2.x (last 2.x release)
- `KAFKA_VER: 3.8.1` - Absolute latest (may have stability concerns)

---

### 2. **GitHub Actions Version** (CRITICAL)

**Current:**
```yaml
- uses: actions/checkout@v2  # DEPRECATED
```

**Updated:**
```yaml
- uses: actions/checkout@v4
```

**Why:**
- `actions/checkout@v2` is deprecated by GitHub
- Will eventually stop working
- v4 has performance and security improvements

---

### 3. **OpenResty Version** (RECOMMENDED)

**Current:**
```yaml
OPENRESTY_VER: 1.19.9.1  # From 2021
```

**Updated:**
```yaml
OPENRESTY_VER: 1.25.3.1  # Latest stable
```

**Why:**
- Security fixes
- Bug fixes
- Better LuaJIT performance
- Not strictly required but good practice

---

### 4. **Kafka Message Format Configuration** (NEW)

**Add this line** to Kafka server configuration (after line 56):
```yaml
sudo sed -i '$alog\.message\.format\.version=2.4' /usr/local/kafka/config/server.properties
```

**Why:**
- Explicitly allows MessageSet format (legacy)
- Ensures compatibility with our limitation (RecordBatch encoding not supported)
- Prevents test failures if Kafka 3.x defaults to newer format

---

## Testing Impact

### Current Tests That Will Pass

✅ All existing tests (TEST 1-10) - Work with both Kafka 2.4 and 3.x
✅ TEST 11 (API v4) - New test, works with MessageSet format
✅ TEST 12 (API v5) - New test, works with MessageSet format

### Why Tests Still Work

1. **MessageSet Format Accepted**: Kafka 3.x accepts MessageSet format (magic byte 0/1) for backward compatibility
2. **No RecordBatch Requirement**: By setting `log.message.format.version=2.4`, we ensure legacy format is accepted
3. **API Version Negotiation**: Client negotiates highest mutually supported API version

---

## Implementation Options

### Option 1: Conservative Update (Recommended for Stability)

Update only critical items:
```yaml
env:
  KAFKA_VER: 2.8.2       # Last 2.x release
  SCALA_VER: 2.13        # Required for 2.8.x
  OPENRESTY_VER: 1.19.9.1  # Keep existing
```

Update GitHub Actions:
```yaml
- uses: actions/checkout@v4
```

**Pros:**
- Minimal change
- Well-tested Kafka version
- Low risk

**Cons:**
- Still using older Kafka (but newer than 2.4.0)

---

### Option 2: Modern Update (Recommended for Future-Proofing)

Update everything:
```yaml
env:
  KAFKA_VER: 3.6.1       # Latest stable 3.x
  SCALA_VER: 2.13        # Required for 3.x
  OPENRESTY_VER: 1.25.3.1  # Latest stable
```

**Pros:**
- Tests with modern Kafka
- Better future compatibility
- Security updates

**Cons:**
- More changes at once
- Slightly higher risk

---

### Option 3: Bleeding Edge (For Advanced Testing)

```yaml
env:
  KAFKA_VER: 3.8.1       # Absolute latest
  SCALA_VER: 2.13
  OPENRESTY_VER: 1.25.3.1
```

**Pros:**
- Tests latest Kafka features
- Finds issues early

**Cons:**
- May have stability issues
- Could fail for reasons unrelated to code

---

## Migration Steps

### Step 1: Update Workflow File

Replace `.github/workflows/test.yaml` with the updated version:

```bash
cp .github/workflows/test-updated.yaml .github/workflows/test.yaml
```

Or manually apply the changes listed above.

### Step 2: Test Locally (Optional)

If you have Docker, test the workflow locally:

```bash
# Start Kafka 3.6.1
docker run -d --name kafka-test \
  -e KAFKA_BROKER_ID=1 \
  -e KAFKA_LISTENERS=PLAINTEXT://localhost:9092 \
  -p 9092:9092 \
  confluentinc/cp-kafka:7.5.0

# Run tests
make test
```

### Step 3: Commit and Push

```bash
git add .github/workflows/test.yaml
git commit -m "Update CI workflow: Kafka 3.6.1, actions/checkout@v4"
git push
```

### Step 4: Monitor CI

Watch the GitHub Actions run to ensure tests pass.

---

## Potential Issues and Solutions

### Issue 1: Kafka 3.x Download Fails

**Symptom:**
```
wget https://archive.apache.org/dist/kafka/3.6.1/kafka_2.13-3.6.1.tgz
404 Not Found
```

**Solution:**
Check available versions at https://archive.apache.org/dist/kafka/ and use an available version.

### Issue 2: Zookeeper Connection Errors (Kafka 3.x)

**Symptom:**
```
Connection to Zookeeper failed
```

**Solution:**
Kafka 3.x can run without Zookeeper (KRaft mode), but our tests use Zookeeper. The current config should work, but if issues arise, may need to add:

```yaml
sudo sed -i '$ainter\.broker\.protocol\.version=2.8' /usr/local/kafka/config/server.properties
```

### Issue 3: Tests Fail with "UNSUPPORTED_VERSION"

**Symptom:**
```
UNSUPPORTED_VERSION error in tests
```

**Solution:**
Add message format version constraint:
```yaml
sudo sed -i '$alog\.message\.format\.version=2.4' /usr/local/kafka/config/server.properties
```

This is already in the updated workflow.

---

## Validation Checklist

After updating the workflow:

- [ ] GitHub Actions runs without errors
- [ ] Kafka broker starts successfully
- [ ] All existing tests pass (TEST 1-10)
- [ ] New API v4+ tests pass (TEST 11-12)
- [ ] SASL authentication tests pass (TEST 9-10)
- [ ] SSL tests pass (TEST 2)
- [ ] No deprecation warnings from GitHub Actions

---

## Rollback Plan

If issues occur:

### Quick Rollback
```bash
git revert <commit-hash>
git push
```

### Manual Rollback
Restore original values:
```yaml
env:
  KAFKA_VER: 2.4.0
  SCALA_VER: 2.11
  OPENRESTY_VER: 1.19.9.1

steps:
  - uses: actions/checkout@v2  # Keep v2 temporarily
```

---

## Recommendation

**Use Option 2 (Modern Update)** unless you have specific constraints:

1. Update to Kafka 3.6.1 (stable and well-tested)
2. Update to actions/checkout@v4 (required)
3. Update to OpenResty 1.25.3.1 (good security practice)
4. Add message format version setting (ensures compatibility)

This provides the best balance of:
- ✅ Modern tooling
- ✅ Good test coverage
- ✅ Reasonable stability
- ✅ Future-proof

---

## Questions?

**Q: Will this break existing functionality?**
A: No. Tests use MessageSet format which is backward compatible.

**Q: Do I need to update my code?**
A: No. These are CI infrastructure updates only.

**Q: What if tests fail after update?**
A: Check WORKFLOW_UPDATES.md "Potential Issues" section and apply solutions. If all else fails, rollback.

**Q: Can I test this before committing?**
A: Yes, use Docker (see Migration Steps) or create a feature branch and check CI results.
