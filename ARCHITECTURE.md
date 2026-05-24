# UHF-AUS Architecture & Design Document

## System Overview

```
┌──────────────────────────────────────────────────────────────┐
│               OPTIMIZATION TASK                              │
│  (TLS, JSON, DNS, HTTP, LOG)                                 │
│  Input: Original code                                         │
│  Target: Select best HW combination                           │
└─────────────────────┬──────────────────────────────────────┘
                      │
        ┌─────────────▼─────────────┐
        │  EvolutionEngine          │
        │  ・Population: 30 agents   │
        │  ・Generations: 50        │
        │  ・Genetic operators:     │
        │    - Mutation             │
        │    - Crossover            │
        │    - Fitness-based repro  │
        └─────────────┬─────────────┘
                      │
        ┌─────────────▼──────────────────────┐
        │         AUSAgent                   │
        │  ┌────────────────────────────┐    │
        │  │ AgentDNA (32 genes)       │    │
        │  │ Each gene ∈ {0,1,2,3,4,5,6,7} │
        │  └────────────┬───────────────┘    │
        │               │ Maps to strategy   │
        │  ┌────────────▼───────────────┐    │
        │  │ STRATEGIES (8 fixed)       │    │
        │  │ 0→apply_hw1_merkle        │    │
        │  │ 1→apply_hw2_zerocopy      │    │
        │  │ ...                        │    │
        │  │ 7→combine_all              │    │
        │  └────────────┬───────────────┘    │
        │               │ Activates HW       │
        │  ┌────────────▼────────────────┐   │
        │  │ Active Higher Weights      │   │
        │  │ e.g. ['HW1','HW2','HW3']  │   │
        │  └────────────┬────────────────┘   │
        │               │ Applies to code    │
        │  ┌────────────▼────────────────┐   │
        │  │ Simplified Code + HW Tags  │   │
        │  │ ANSWER 1: Code             │   │
        │  │ ANSWER 2: HW list          │   │
        │  └────────────┬────────────────┘   │
        │               │ Evaluate           │
        │  ┌────────────▼────────────────┐   │
        │  │ UHF Metrics                │   │
        │  │ γ = α·(1/(1+δ))            │   │
        │  │   + β·T                    │   │
        │  │   + (1-α-β)·(1-e^-M)      │   │
        │  └────────────┬────────────────┘   │
        │               │ γ ∈ [0,1]         │
        │  ┌────────────▼────────────────┐   │
        │  │ Life/Death Decision        │   │
        │  │ γ ≥ 0.4 → ALIVE ✅         │   │
        │  │ γ < 0.4 → DEAD ☠️          │   │
        │  └────────────┬────────────────┘   │
        │               │                    │
        │  ┌────────────▼────────────────┐   │
        │  │ Store Results              │   │
        │  │ ・gamma history            │   │
        │  │ ・energy_saved_twh         │   │
        │  │ ・HW applied               │   │
        │  └────────────────────────────┘    │
        │                                    │
        └────────────────────────────────────┘
                      │
           ┌──────────▼──────────┐
           │   Next Generation   │
           │   ・Selection       │
           │   ・Crossover       │
           │   ・Mutation        │
           │   ・Reproduction    │
           └─────────────────────┘
```

---

## Component Details

### 1. UHFConfig (Configuration)

```python
@dataclass
class UHFConfig:
    # Weighting for γ formula
    alpha: float = 0.5      # Weight of structure (δ component)
    beta: float = 0.3       # Weight of correctness (T component)
    # Implicitly: (1-α-β) = 0.2 weight on experience (M component)
    
    # Thresholds
    gamma_death: float = 0.4    # Below this → agent dies
    gamma_elite: float = 0.8    # Above this → preserved in next gen
    
    # Population dynamics
    population_size: int = 30          # Agents per generation
    generations: int = 50              # Total iterations
    mutation_rate: float = 0.3         # Gene flip probability
    chaos_injection_threshold: float = 0.15  # When diversity too low
```

**Design Decision**: These are hyperparameters, not data. They define the "rules of the game".

---

### 2. AgentDNA (Genotype)

```python
@dataclass
class AgentDNA:
    genes: np.ndarray = field(default_factory=...)  # Shape: (32,)
    # Each gene ∈ {0,1,2,3,4,5,6,7}
    
    STRATEGIES = {
        0: ("apply_hw1_merkle", "HW1"),
        1: ("apply_hw2_zerocopy", "HW2"),
        2: ("apply_hw3_stateful", "HW3"),
        3: ("apply_hw4_probabilistic", "HW4"),
        4: ("apply_hw5_binary", "HW5"),
        5: ("combine_hw1_hw2", "HW1+HW2"),     # Combinations
        6: ("combine_hw3_hw5", "HW3+HW5"),
        7: ("combine_all", "ALL"),
    }
```

**Why 32 genes?**
- Redundancy: 32 genes selecting from 8 strategies
- Stabilization: Multiple copies reduce noise
- Crossover mixing: More loci = smoother recombination

**How it works**:
```python
genes = [1, 3, 1, 5, 2, 7, 1, 3, ...]  # Example
unique = set(genes) = {1, 3, 5, 2, 7}
active_hws = [STRATEGIES[g] for g in unique]
            = [("apply_hw2_zerocopy", "HW2"),
               ("apply_hw4_probabilistic", "HW4"),
               ("combine_hw1_hw2", "HW1+HW2"),
               ("apply_hw3_stateful", "HW3"),
               ("combine_all", "ALL")]
```

---

### 3. AUSAgent (Individual)

```python
class AUSAgent:
    id: int                                    # Unique identifier
    dna: AgentDNA                              # Genotype
    
    # Phenotype (UHF metrics)
    gamma: float        # Coherence [0,1]
    delta: float        # Complexity [0,2]
    T: float            # Correctness [0,1]
    M: float            # Experience [0,1]
    
    # State
    alive: bool         # γ ≥ gamma_death?
    generation: int     # Birth generation
    
    # Outputs (TWO ANSWERS)
    answer_1_simplified_code: str       # Modified source
    answer_2_higher_weights: List[str]  # Applied HW
    
    # History
    gamma_history: List[float]    # γ over time
    energy_saved_twh: float       # Estimated (MODEL)
```

**Lifecycle**:
```
CREATION (random DNA)
    ↓
EVALUATION (compute γ)
    ├─ γ ≥ 0.4 → ALIVE ✅
    └─ γ < 0.4 → DEAD ☠️
    ↓
SELECTION (elite survive)
    ↓
REPRODUCTION (mutate + crossover)
    ↓
NEXT GENERATION
```

---

### 4. Higher Weights Library

```python
HIGHER_WEIGHTS = {
    "HW1": HigherWeight(
        id="HW1",
        name="Merkle Proof Instead of Data",
        formula="proof(data) << data",
        description="...",
        applicability=0.87,        # Theoretical success rate
        energy_impact_twh=69.2,    # ⚠️ PRIOR (not measured)
        domains=["TLS", "DNS", "PKI", ...],
        code_template="..."
    ),
    # ... HW2-HW5
}
```

**Key property**: ALL predefined. Evolution SELECTS, doesn't CREATE.

---

### 5. UHF Coherence Metric (γ)

The core formula:

```
γᵢ = α·(1/(1+δᵢ)) + β·Tᵢ + (1-α-β)·(1-e^(-Mᵢ))
```

Where:
- **α** = weight of structure (δ component)
- **β** = weight of correctness (T component)  
- **(1-α-β)** = weight of experience (M component)

**Components breakdown**:

#### 5.1 δ (Complexity)

```python
def complexity_delta(original: str, simplified: str) -> float:
    """
    Measure: simplified_length / original_length
    
    ⚠️ CAVEAT: String length is not runtime complexity!
    """
    orig_len = len(original)
    simp_len = len(simplified)
    delta = simp_len / orig_len if orig_len > 0 else 1.0
    return np.clip(delta, 0.01, 2.0)
```

**Examples**:
```
Original: 1000 bytes
Simplified to 800 bytes → δ = 0.8 → (1/(1+0.8)) ≈ 0.556
Simplified to 600 bytes → δ = 0.6 → (1/(1+0.6)) ≈ 0.625
Simplified to 1500 bytes → δ = 1.5 → (1/(1+1.5)) ≈ 0.4
```

#### 5.2 T (Correctness)

```python
def _measure_ast_preservation(original: str, simplified: str) -> float:
    """
    Measure: % of class/function names preserved
    
    ⚠️ CAVEAT: AST-only, no semantic validation!
    """
    orig_tree = ast.parse(original)
    simp_tree = ast.parse(simplified)
    
    orig_names = {n.name for n in walk(orig_tree) 
                  if isinstance(n, (ClassDef, FunctionDef))}
    simp_names = {n.name for n in walk(simp_tree) 
                  if isinstance(n, (ClassDef, FunctionDef))}
    
    if not orig_names:
        return 1.0
    
    T = len(orig_names & simp_names) / len(orig_names)
    return np.clip(T, 0, 1)
```

**Examples**:
```
Original classes: {Logger, HTTPRequest, DNSResolver}
Simplified keeps: {Logger, HTTPRequest}
T = 2/3 ≈ 0.667

Original classes: {Logger, HTTPRequest, DNSResolver}
Simplified keeps: {Logger, HTTPRequest, DNSResolver, NewHelper}
T = 3/3 = 1.0  (new classes OK, only care about preservation)
```

#### 5.3 M (Experience/Energy)

```python
def evaluate(self, target, metrics):
    if applied_hw:
        base_energy = sum(
            HIGHER_WEIGHTS[hw].energy_impact_twh * 
            HIGHER_WEIGHTS[hw].applicability
            for hw in applied_hw
        )
        M = min(1.0, base_energy / target.energy_twh)
    else:
        M = 0.0
```

**Why normalize by target.energy_twh?**
- Prevents >100% "savings"
- Makes M ∈ [0,1] naturally
- Reflects: "how much of this task's energy could we save?"

**Example**:
```
Task: TLS (48 TWh)
Applied: HW1+HW2+HW3
HW1 impact: 69.2 × 0.87 ≈ 60.2 TWh
HW2 impact: 52.5 × 0.79 ≈ 41.5 TWh
HW3 impact: 40.1 × 0.83 ≈ 33.3 TWh
Sum: 135 TWh (theoretical max)
M = min(1.0, 135 / 48) = 1.0 (capped)

Task: LOG (18 TWh)
Applied: HW4+HW5
HW4 impact: 23.9 × 0.71 ≈ 17.0 TWh
HW5 impact: 22.4 × 0.92 ≈ 20.6 TWh
Sum: 37.6 TWh (over budget!)
M = min(1.0, 37.6 / 18) = 1.0 (capped)
```

#### 5.4 Final γ Computation

```python
gamma = (
    0.5 * (1 / (1 + delta)) +     # 50% weight on structure
    0.4 * T +                      # 40% weight on correctness
    0.1 * (1 - np.exp(-M))         # 10% weight on experience
)
```

**Why exponential for M?**
```
1 - e^(-M):
  M=0   → 1 - e^0 = 0
  M=0.1 → 1 - e^(-0.1) ≈ 0.095
  M=0.5 → 1 - e^(-0.5) ≈ 0.393
  M=1.0 → 1 - e^(-1) ≈ 0.632
  M=2.0 → 1 - e^(-2) ≈ 0.865
```

Diminishing returns on energy. Makes sense: first HW gives big boost, next ones less.

---

### 6. EvolutionEngine (Population Dynamics)

```
Generation 0:
  Create 30 random agents
  ↓
For gen = 0 to 49:
  1. EVALUATION: Each agent evaluates on task
     - Compute δ, T, M
     - Compute γ = f(δ, T, M)
     - Decide: alive? (γ ≥ 0.4)
  
  2. STATS: Record population metrics
     - Alive count
     - Mean/best γ
     - Best HW applied
  
  3. SELECTION: Keep elite (γ ≥ 0.8)
  
  4. REPRODUCTION: Fill next generation
     - Elite: 100% copy
     - Survivors: Mutate or crossover
     - If diversity too low: chaos injection
```

**Reproduction logic**:

```python
def _reproduce(self):
    survivors = [a for a in self.population if a.alive]
    elite = [a for a in survivors if a.gamma >= 0.8]
    
    new_population = []
    
    # 1. Keep elite (no changes)
    for agent in elite[:max(1, len(elite))]:
        new_population.append(agent.clone())
    
    # 2. Fill rest with offspring
    while len(new_population) < population_size:
        if len(survivors) >= 2 and random() < 0.4:
            # Crossover (sexual reproduction)
            p1, p2 = random.choices(survivors[:5], k=2)
            child = p1.crossover(p2)
        else:
            # Mutation (asexual reproduction)
            parent = random.choice(survivors[:5])
            child = parent.clone()
            child.mutate()
        
        new_population.append(child)
```

**Key:** `survivors[:5]` biases selection toward best agents (top 5), while not excluding worse ones. This maintains diversity.

---

## Information Flow Example

### Scenario: Evolving for TLS Handshake

```
Task: TLS (48 TWh)
Code: TLSHandshake class with handshake() method

AGENT #7:
  DNA: [1, 3, 1, 5, 2, 1, ...]
  Active strategies: {1, 3, 5, 2} → HW2, HW4, HW1+HW2, HW3
  Unique HW: [HW1, HW2, HW3, HW4]
  
  Apply HW:
    Original code (540 bytes)
    + HW1 comment (120 bytes)
    + HW2 comment (110 bytes)
    + HW3 comment (100 bytes)
    + HW4 comment (95 bytes)
    = Simplified code (865 bytes)
  
  Evaluate:
    δ = 865 / 540 ≈ 1.60
      → (1/(1+1.60)) ≈ 0.385
    
    T = classes preserved: {TLSHandshake} / {TLSHandshake}
      = 1.0
    
    M = (69.2×0.87 + 52.5×0.79 + 40.1×0.83 + 23.9×0.71) / 48
      = (60.2 + 41.5 + 33.3 + 17.0) / 48
      = min(152 / 48, 1.0) = 1.0
    
    γ = 0.5 × 0.385 + 0.4 × 1.0 + 0.1 × (1 - e^(-1.0))
      = 0.193 + 0.4 + 0.0632
      ≈ 0.656
  
  Status: ALIVE ✅ (γ ≥ 0.4)
  
  ANSWERS:
    answer_1: "# HW1: ...\n# HW2: ...\n[original code]"
    answer_2: ['HW1', 'HW2', 'HW3', 'HW4']
    energy_saved_twh: min(152, 48) = 48 TWh (capped)
```

---

## Genetic Operators

### Mutation

```python
def mutate(self, rate: float = 0.3):
    """Flip genes with probability `rate`"""
    mask = np.random.random(self.size) < rate
    self.genes[mask] = np.random.randint(0, 8, mask.sum())
```

With `rate=0.3` and `size=32`:
- Expected flips: 32 × 0.3 ≈ 9.6 genes per agent per generation
- This is aggressive (31% of genome), but prevents stagnation

### Crossover

```python
def crossover(self, other: 'AgentDNA') -> 'AgentDNA':
    """Single-point crossover"""
    child = AgentDNA(genes=self.genes.copy())
    mask = np.random.random(self.size) > 0.5
    child.genes[mask] = other.genes[mask]  # 50% from each parent
    return child
```

Example:
```
Parent 1: [1, 2, 3, 4, 5, 6, 7, 0, ...]
Parent 2: [7, 6, 5, 4, 3, 2, 1, 0, ...]
Mask:     [T, F, T, F, T, F, T, F, ...]
Child:    [1, 6, 3, 4, 5, 2, 7, 0, ...]  (alternating from parents)
```

---

## Key Design Decisions

### 1. Why 8 STRATEGIES (not infinite)?

**Why not grammar-based synthesis?**
- Synthesis would claim "emergent discovery" ✗
- Would require semantic validation ✗
- Harder to explain results ✗

**Why 8?**
- 5 core HW (HW1-HW5)
- 2 combinations (HW1+HW2, HW3+HW5)
- 1 "apply all"
- Constrained space makes results reproducible ✓

### 2. Why 32 Genes?

- 32 = 4 × 8 (redundancy for stability)
- Small enough to mutate quickly
- Large enough to carry signal
- Trade-off: mutation rate vs convergence speed

### 3. Why γ ≥ 0.4 as Death Threshold?

Looking at formula with default params:
```
δ = 1 (no simplification), T = 1 (perfect), M = 0 (no energy)
γ = 0.5×(1/2) + 0.4×1 + 0.1×0 = 0.25 + 0.4 = 0.65

δ = 1, T = 0.5, M = 0:
γ = 0.25 + 0.2 + 0 = 0.45

δ = 2, T = 1, M = 0:
γ = 0.5×(1/3) + 0.4 + 0 ≈ 0.57

δ = 1, T = 0, M = 0:
γ = 0.25 + 0 + 0 = 0.25  ← DIES
```

So 0.4 is roughly the "zero energy, broken code" region. Meaningful cutoff.

### 4. Why Exponential for M?

```
Motivation: Energy impact follows S-curve
  - First optimization (HW1) saves 50% of potential
  - Second (HW2) saves 20% of remaining
  - Third (HW3) saves 10% of remaining
  
This is NOT true in the code (all are independent),
but (1 - e^-M) gives diminishing returns feel.
```

---

## Testing and Debugging

### Run with Seed for Reproducibility

```python
import random, numpy as np

random.seed(42)
np.random.seed(42)
results, _ = run_simulation()
# Same results every time ✓
```

### Trace Single Agent

```python
agent = AUSAgent(config)
agent.dna.genes = np.array([1, 1, 1, 1, 1, 1, 1, 1, ...])  # All HW2

result = agent.evaluate(TLS_TARGET, UHFMetrics())

print(f"HW applied: {agent.answer_2_higher_weights}")
print(f"Simplified code length: {len(agent.answer_1_simplified_code)}")
print(f"γ = {agent.gamma:.3f}")
print(f"δ = {agent.delta:.3f}")
print(f"T = {agent.T:.3f}")
print(f"M = {agent.M:.3f}")
print(f"Alive: {agent.alive}")
```

### Profile Evolution

```python
import time

engine = EvolutionEngine(config)
start = time.time()
results = engine.run(TLS_TARGET, generations=50)
elapsed = time.time() - start

print(f"Time: {elapsed:.2f}s")
print(f"Agents/sec: {config.population_size * 50 / elapsed:.0f}")
print(f"Best γ: {results['best_gamma']:.3f}")
```

Typically: 30 agents × 50 gens × 5 tasks ≈ 5-10 seconds on modern CPU.

---

## Conclusion

UHF-AUS is a **principled selector** with:
- Clear mathematical framework (γ formula)
- Transparent design choices (8 strategies, 32 genes)
- Honest limitations (PRIOR energy, AST-only correctness)
- Reproducible results (seed-based)

It's NOT a black-box heuristic, and that's the point.

---

**Version**: 3.1  
**Last Updated**: 2024  
**Status**: Framework-ready, not production-ready
