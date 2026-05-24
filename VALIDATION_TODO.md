# Validation & Real-World Testing Roadmap

**Status**: UHF-AUS v3.1 is a **framework**, not production software.

This checklist describes how to graduate it to real validation.

---

## Phase 1: Energy Profiling (2-3 weeks)

### 1.1 Hardware Setup

- [ ] Access Intel Xeon/Core with RAPL support (Sandy Bridge+) OR
- [ ] Access AMD EPYC with RAPL support (Zen+) OR
- [ ] Access ARM with energy counters (Apple M1/M2 or Qualcomm)
- [ ] Stable thermal baseline (room temp 20-22°C, no thermal throttling)
- [ ] Dedicated machine (no concurrent workloads)

### 1.2 RAPL Profiling Tools

**Option A: turbostat (Intel)**
```bash
sudo turbostat --interval 1000 --show Package,Core \
  --Summary --quiet -- python aus_original_code.py

# Output: Energy in Joules every 1 second
# Compare: original vs simplified
```

**Option B: perf (Linux, any CPU)**
```bash
sudo perf stat -e power/energy-cores/,power/energy-pkg/ \
  -a python aus_original_code.py

# Output: Energy counters for code execution
```

**Option C: custom profiler**
```python
# If no RAPL available, measure as proxy:
import time
start_time = time.perf_counter()
result = run_code()
elapsed = time.perf_counter() - start_time
# Energy ≈ (CPU_freq_GHz × clock_cycles × voltage) × elapsed
```

### 1.3 Protocol-Specific Profiling

**For TLS**: 
```bash
# Capture handshake traffic
tcpdump -i eth0 'tcp port 443' -w tls.pcap

# Parse: measure bytes per handshake
tshark -r tls.pcap -T fields -e frame.len | stats
```

**For HTTP Headers**:
```bash
curl -I https://example.com 2>&1 | grep -E "^[A-Z-]+:" | \
  awk '{print length}' | sum  # Total header bytes
```

**For DNS**:
```bash
dig @8.8.8.8 google.com +stats  # Query + response sizes
```

**For JSON**:
```python
import json
original = json.dumps({"user": {"id": 1}})
print(len(original.encode()))  # Original bytes
```

**For Logging**:
```bash
# Generate typical logs, measure disk I/O
logger --tag testapp --priority user.info "Test message"
# Monitor: iostat, vmstat, or perf c2c
```

### 1.4 Baseline Measurements

For each task:

```
╔═══════════════════════════════════════════════╗
║ BASELINE: TLS Handshake                       ║
╠═══════════════════════════════════════════════╣
║ Energy (RAPL):     [___] Joules per op        ║
║ Time (perf):       [___] milliseconds         ║
║ Bytes transferred: [___] bytes per handshake ║
║ CPU cycles:        [___] cycles               ║
║ Cache misses:      [___] % of accesses        ║
║ Peak memory:       [___] MB                   ║
╚═══════════════════════════════════════════════╝
```

Repeat 5 times, take median (exclude min/max for outliers).

---

## Phase 2: Semantic Correctness (2-3 weeks)

### 2.1 Test Input/Output Comparison

For each task, create a test harness:

```python
# test_harness.py

def test_correctness(original_code, simplified_code, test_cases):
    """
    Execute both, compare outputs.
    
    ⚠️  Only valid for PURE functions.
    For side-effectful code (logging, network), 
    compare effects instead.
    """
    
    for i, test_input in enumerate(test_cases):
        orig_result = execute(original_code, test_input)
        simp_result = execute(simplified_code, test_input)
        
        if orig_result != simp_result:
            print(f"MISMATCH at test {i}:")
            print(f"  Original: {orig_result}")
            print(f"  Simplified: {simp_result}")
            return False
    
    return True  # All tests passed
```

### 2.2 Task-Specific Validators

**TLS Handshake**:
```python
# Verify: handshake produces valid session key
def validate_tls(original_code, simplified_code):
    # Both should produce same-length key (32 bytes)
    # Both should complete in <1 second
    pass
```

**JSON Parsing**:
```python
# Verify: extracted fields match
test_json = '{"user":{"id":123},"amount":99.99}'

orig_obj = json.loads(test_json)
orig_id = orig_obj['user']['id']

# Simplified must extract same ID
assert orig_id == 123
```

**DNS Resolution**:
```python
# Verify: resolved IPs match expected
domains = ['google.com', 'github.com']

orig_results = original_resolver.resolve_batch(domains)
simp_results = simplified_resolver.resolve_batch(domains)

for domain in domains:
    assert orig_results[domain] == simp_results[domain]
```

**HTTP Headers**:
```python
# Verify: header count/size consistency
orig_headers = original_request.headers
simp_headers = simplified_request.headers

# Both should parse as valid HTTP
assert parse_http(orig_headers) is not None
assert parse_http(simp_headers) is not None

# Key headers must be present
assert 'Content-Type' in simp_headers
```

**Logging**:
```python
# Verify: all ERROR/CRITICAL logs captured
# (DEBUG/TRACE sampling expected due to HW4)

logger = Logger()

logger.error("Critical error")
logger.debug("Debug info")

# Even with sampling, ERROR should be captured
assert "Critical error" in logger.logs
```

### 2.3 Regression Test Suite

```bash
# Run test suite before and after simplification
pytest test_original.py    # Baseline
pytest test_simplified.py  # After HW applied

# Both should pass 100%
# If simplified fails: mark T = 0 for that agent
```

---

## Phase 3: Ablation Studies (3-4 weeks)

### 3.1 Per-HW Impact Analysis

Isolate each HW to measure individual contribution:

```python
def ablation_study(target):
    """
    Compare energy savings with each HW individually.
    """
    
    results = {}
    
    # Baseline: no HW applied
    baseline_energy = profile(target.original_code)
    
    # HW1 only
    with_hw1 = profile(apply_hw(target.original_code, "HW1"))
    results["HW1"] = baseline_energy - with_hw1
    
    # HW2 only
    with_hw2 = profile(apply_hw(target.original_code, "HW2"))
    results["HW2"] = baseline_energy - with_hw2
    
    # ... repeat for HW3, HW4, HW5
    
    # Combinations
    with_hw12 = profile(apply_hw(target.original_code, "HW1", "HW2"))
    results["HW1+HW2"] = baseline_energy - with_hw12
    
    return results
```

**Expected output**:
```
Task: TLS
  HW1 alone:    +15% energy savings
  HW2 alone:    +22% energy savings
  HW1+HW2:      +32% (not 15+22, due to interaction)
  HW1+HW2+HW3:  +41% (diminishing returns)
```

### 3.2 Interaction Matrix

```
╔═════════════════════════════════════════════════╗
║ HW Combinations: Energy Savings (%)             ║
╠══════╦════╦════╦════╦════╦════╦═══════╦═══════╣
║  HW  ║HW1 ║HW2 ║HW3 ║HW4 ║HW5 ║HW1+2  ║HW3+5  ║
╠══════╬════╬════╬════╬════╬════╬═══════╬═══════╣
║ TLS  ║ 15 ║ 22 ║ 18 ║  5 ║  8 ║  32   ║  22   ║
║ JSON ║  2 ║ 35 ║  0 ║  0 ║ 18 ║  38   ║  18   ║
║ DNS  ║ 25 ║  8 ║  0 ║ 12 ║  0 ║  30   ║   0   ║
║ HTTP ║  0 ║  5 ║ 28 ║  0 ║  8 ║   5   ║  35   ║
║ LOG  ║  0 ║  0 ║  0 ║ 18 ║ 40 ║   0   ║  50   ║
╚══════╩════╩════╩════╩════╩════╩═══════╩═══════╝
```

This shows:
- HW5 is best for LOG (40%)
- HW3+HW5 dominates HTTP (35%)
- HW interactions vary by task

---

## Phase 4: Multi-Seed Statistics (1-2 weeks)

### 4.1 Run 100 Seeds

```python
import numpy as np
from scipy import stats

results_all = []

for seed in range(100):
    np.random.seed(seed)
    random.seed(seed)
    
    results = run_simulation()
    results_all.append(results)

# Extract metric across seeds
gammas = [r['best_gamma'] for r in results_all]
hw_frequencies = {}
for r in results_all:
    for hw in r['applied_hw']:
        hw_frequencies[hw] = hw_frequencies.get(hw, 0) + 1
```

### 4.2 Compute Confidence Intervals

```python
gamma_mean = np.mean(gammas)
gamma_std = np.std(gammas)

# 95% CI
ci_lower = gamma_mean - 1.96 * (gamma_std / np.sqrt(100))
ci_upper = gamma_mean + 1.96 * (gamma_std / np.sqrt(100))

print(f"γ = {gamma_mean:.3f} ± [{ci_lower:.3f}, {ci_upper:.3f}]")

# HW dominance
hw_freq_pct = {hw: (count/100)*100 for hw, count in hw_frequencies.items()}
print("\nHW Dominance (% of runs):")
for hw, pct in sorted(hw_freq_pct.items(), key=lambda x: x[1], reverse=True):
    print(f"  {hw}: {pct:.1f}%")
```

### 4.3 Convergence Analysis

```python
# For each seed, track γ over generations
gamma_trajectories = [
    r['gamma_trajectory'] for r in results_all
]

# Plot convergence
import matplotlib.pyplot as plt

for trajectory in gamma_trajectories:
    plt.plot(range(len(trajectory)), trajectory, alpha=0.1)

plt.xlabel('Generation')
plt.ylabel('Best γ')
plt.title('Convergence over 100 seeds')
plt.show()

# Compute median trajectory
median_trajectory = np.median(gamma_trajectories, axis=0)
plt.plot(median_trajectory, 'r-', linewidth=2)
plt.show()
```

---

## Phase 5: Train/Test Split (1 week)

### 5.1 Divide Tasks into Sets

```
Train Set: TLS, JSON, DNS    (evolve on these)
Test Set:  HTTP, LOG          (validate on these)
```

### 5.2 Evolve on Train

```python
config = UHFConfig(...)
engine = EvolutionEngine(config)

# Evolve ONLY on train tasks
for target in TRAIN_TARGETS:  # TLS, JSON, DNS
    result = engine.run(target)
    # engine.best_ever now adapted to train distribution
```

### 5.3 Validate on Test

```python
# Test generalization: apply same HW to unseen tasks
test_results = []
for target in TEST_TARGETS:  # HTTP, LOG
    best_hws = engine.best_ever.answer_2_higher_weights
    
    # Apply same HW to new task
    simplified, applied = apply_hws_to_code(target.original_code, best_hws)
    
    # Measure: does it help on new task?
    energy_saved = profile(target.original_code) - profile(simplified)
    
    test_results.append({
        'task': target.id,
        'train_hws': best_hws,
        'energy_saved': energy_saved,
    })
```

### 5.4 Compare Generalization

```
╔══════════════════════════════════════════╗
║ Generalization: Train vs Test            ║
╠════════╦═════════════╦═════════════════╣
║ Task   ║ Train Perf  ║ Test Perf (%)   ║
╠════════╬═════════════╬═════════════════╣
║ HTTP   ║ N/A (test)  ║ +28% ✅        ║
║ LOG    ║ N/A (test)  ║ +35% ✅        ║
║ (avg)  ║ N/A         ║ +31.5% ✅      ║
╚════════╩═════════════╩═════════════════╝
```

If test performance >70% of train performance: generalization is good ✓

---

## Phase 6: Comparison Benchmarks (2-3 weeks)

### 6.1 Baseline Comparisons

Compare UHF-AUS selection against:

1. **Random selection** (baseline)
   ```python
   random_hw = random.sample(HIGHER_WEIGHTS.keys(), k=2)
   energy_saved_random = measure_energy(apply_hws(code, random_hw))
   ```

2. **Greedy by applicability** (heuristic)
   ```python
   sorted_hw = sorted(HIGHER_WEIGHTS.items(), 
                      key=lambda x: x[1].applicability, 
                      reverse=True)[:3]
   energy_saved_greedy = measure_energy(apply_hws(code, [h[0] for h in sorted_hw]))
   ```

3. **LLVM compiler optimizations** (existing solution)
   ```bash
   clang -O3 original.c -o original_opt
   measure_energy(original_opt)
   ```

4. **PyPy JIT** (existing Python solution)
   ```bash
   pypy original.py  # Run with PyPy
   measure_energy(pypy)
   ```

### 6.2 Results Table

```
╔═════════════════════════════════════════════════╗
║ Method Comparison: Energy Savings %             ║
╠══════════════════╦═════════════════════════════╣
║ Method           ║ Energy Savings (%)          ║
╠══════════════════╬═════════════════════════════╣
║ Random           ║ +2.3% ± 4.1%  (baseline)   ║
║ Greedy (applicab)║ +8.7% ± 2.1%               ║
║ LLVM -O3         ║ +34.2% (C code only)       ║
║ PyPy JIT         ║ +18.5% (Python runtime)    ║
║ UHF-AUS (v3.1)   ║ +28.1% ± 3.5% ← **BEST**  ║
╚══════════════════╩═════════════════════════════╝
```

---

## Phase 7: Documentation & Release (1 week)

### 7.1 Finalize Reports

- [ ] Energy profiling report (RAPL data + graphs)
- [ ] Correctness validation report (test coverage %)
- [ ] Ablation study report (per-HW contributions)
- [ ] Multi-seed statistics (confidence intervals)
- [ ] Generalization study (train/test split results)
- [ ] Benchmark comparison (vs baselines)

### 7.2 Update README

```markdown
# UHF-AUS v3.2: Validated Release

## Real-World Results

- **Energy Savings**: 28.1% ± 3.5% (N=100 seeds)
- **Correctness**: 100% (all test cases pass)
- **Generalization**: 31.5% on unseen tasks
- **Ablation**: HW5 contributes 12%, HW3 contributes 8%, ...
- **vs Baseline**: 2.8x better than random selection

## Validation Status

- [x] RAPL profiling completed
- [x] Semantic validation (100 test cases)
- [x] Ablation studies (per-HW analysis)
- [x] Multi-seed statistics (100 runs)
- [x] Train/test generalization
- [x] Benchmark vs competitors

**Production Ready**: YES ✅ (with caveats below)
```

### 7.3 Caveats for Users

```markdown
## Important Limitations

1. **Hardware-specific**: Energy savings measured on Intel Xeon.
   May differ on ARM or AMD by ±5-15%.

2. **Workload-specific**: Results on synthetic tasks.
   Real-world workloads may have different patterns.

3. **Code context**: Changes assume greenfield optimization.
   Existing LLVM optimizations not removed.

4. **Overhead not included**: Measurement doesn't account for
   - Recompilation time
   - Testing burden
   - Maintenance cost

## When to Use

✅ Use if:
   - You control source code
   - Energy profiling is critical
   - You can validate correctness

❌ Don't use if:
   - Black-box third-party code
   - Correctness uncertain
   - Time-to-market critical
```

---

## Success Criteria

Mark complete only if:

- [ ] Energy savings ≥ 20% (realistic, not inflated)
- [ ] Correctness ≥ 95% (few bugs)
- [ ] Generalization ≥ 60% of train performance
- [ ] Ablation shows clear per-HW contributions
- [ ] Multi-seed CI width < 5% of mean
- [ ] Benchmark competitive (top 3 vs random/greedy/existing)

---

## Timeline Estimate

| Phase | Duration | Effort |
|-------|----------|--------|
| 1. Energy | 2-3 weeks | 1 person |
| 2. Correctness | 2-3 weeks | 1 person |
| 3. Ablation | 3-4 weeks | 1 person + compute |
| 4. Multi-seed | 1-2 weeks | compute |
| 5. Train/test | 1 week | 1 person |
| 6. Benchmarks | 2-3 weeks | 2 people |
| 7. Docs | 1 week | 1 person |
| **TOTAL** | **12-19 weeks** | **~5 person-weeks** |

---

## Checklist Before Production Deploy

- [ ] All profiling complete with reproducible numbers
- [ ] Semantic correctness validated on >100 test cases
- [ ] Ablation shows HW contributions are isolated
- [ ] Multi-seed stats show narrow confidence intervals
- [ ] Generalization test passed (train/test split)
- [ ] Comparison vs baselines shows competitive advantage
- [ ] Edge cases documented (where it fails)
- [ ] Performance overhead quantified
- [ ] README updated with real data
- [ ] Code reviewed by 2+ independ experts
- [ ] Repository public with MIT license
- [ ] Citation & reproducibility info clear

---

**Version**: Validation Roadmap v1.0  
**Last Updated**: 2024  
**Status**: Ready to start Phase 1 ✅
