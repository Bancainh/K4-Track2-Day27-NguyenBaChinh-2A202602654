# Reflection (≤1 page)

### Which fault types were hardest to catch, and why?

1. **Subtle-tier distribution shifts, volume drops/spikes, and runtime anomalies:**
   - In earlier phases, faults predominantly presented as large, obvious deviations well beyond the calibrated $3\sigma$ baseline bounds (such as massive volume surges or extreme latency delays). These were straightforward to flag with high confidence.
   - However, the harder stream introduced subtle-magnitude anomalies that sit directly within the natural empirical envelope of clean traffic. Observed metric ranges showed substantial overlap between normal variations and subtle anomalies—for example, micro-fluctuations in batch sizes, slight runtime variations, or modest distribution shifts.
   - The scoring formulation heavily penalizes false alarms ($-0.3 \cdot \text{FPR}$ versus $+0.5 \cdot \text{TPR}$). Because clean traffic comprises the majority of stream events, tightening static thresholds into normal variance intervals triggers multiple false positives on clean batches. Mathematically, avoiding false alarms outweighs guessing ambiguous marginal cases, making point-in-time thresholding ill-suited for subtle anomalies that fall within clean dispersion.

2. **Contract checkpoints and feature skew were the most reliable:**
   - Contract checkpoints provided deterministic, binary signals: schema hash mismatches, corrupted data types, and explicit SLA freshness delivery breaches. These achieved perfect recall with zero false alarms.
   - Feature materializations proved reliable once the drift threshold was calibrated slightly above empirical sample variance (using a $1.5\times$ baseline buffer to absorb small-batch sampling noise), effectively separating actual feature skew from normal drift without false positives.

---

### What would you change about your cost/coverage tradeoff, if you had another pass?

1. **Phase-Adaptive Dynamic Budgeting:**
   - While the private phase provided sufficient credit headroom for full single-pass coverage ($300.0$ credits spent against a $320.0$ budget, $0.0$ overage), tighter budgets (such as in public phase) penalize redundant checks.
   - If given another pass, we would implement adaptive budgeting via `ctx.tools.budget_remaining()`. When remaining budget falls below safe thresholds, we would throttle or skip expensive $2.0$-credit checks (`feature_drift`, `embedding_drift`) on alternating cycles while preserving high-yield, low-cost $1.0$-credit checks (`batch_profile`, `lineage_graph_slice`).

2. **Stateful Change-Point and Cumulative Sum (CUSUM) Detectors:**
   - Single-event inspection is fundamentally blind to subtle perturbations within $\pm 3\sigma$. Leveraging `ctx.state` (which incurs zero RPC credit cost), we would accumulate sequential statistics across micro-batches using algorithms like CUSUM or Page-Hinkley tests.
   - By tracking running means and cumulative residual sums over time, a stateful detector can reliably uncover sustained subtle shifts while keeping point-in-time false alarm rates at zero.
