# Reflection (≤1 page)

### Which fault types were hardest to catch, and why?

1. **Subtle-tier distribution shifts, volume drops/spikes, and runtime anomalies:**
   - In the Practice and Public phases, faults predominantly manifest as significant deviations far beyond baseline thresholds (e.g., volume spikes > 800 rows against a baseline max of 561, or runtime durations > 7500 ms against a max of 5135 ms). These are cleanly caught by calibrated $3\sigma$ bounds.
   - In the Private phase, however, subtle-tier faults were intentionally injected within the natural empirical envelope of clean traffic (for example, subtle runtime anomalies with durations of 4,481–4,739 ms while clean runs naturally vary up to 4,805 ms; subtle volume drops with 469–474 rows while clean batches range down to 452 rows).
   - Because the scoring function heavily penalizes false alarms ($-0.3 \cdot \text{FPR}$ vs $+0.5 \cdot \text{TPR}$), tightening static bounds into the normal variance interval triggers multiple false positives on clean batches. In a 200-event stream with 146 clean events, triggering even a few false positives quickly offsets the small TPR gain from guessing subtle instances.
   - Consequently, static point-in-time thresholds cannot separate subtle faults that completely overlap with clean variance without incurring prohibitive false-alarm penalties.

2. **Feature skew and Contract checkpoints were the most reliable:**
   - Contract checkpoints (`contracts` pillar) provided deterministic signals: schema hash mismatches, data type corruptions (`actual_amount_type != "float"`), and explicit SLA delivery breaches (`freshness_delay_min > declared_sla`). These yielded 100% TPR with 0% FPR across all phases.
   - Feature materializations (`ai_infra` pillar) exhibited clean empirical separation: clean sampling variance occasionally reached $\approx 0.47\sigma$ (exceeding the initial $0.4095\sigma$ baseline and causing public-phase false alarms), while true skews were consistently $\ge 0.97\sigma$. Setting a robust threshold at $\max(1.5 \times \text{baseline}, 0.70\sigma)$ eliminated all false alarms while preserving 100% recall.

---

### What would you change about your cost/coverage tradeoff, if you had another pass?

1. **Phase-Adaptive Budget Allocation:**
   - In the Private phase, the 320.0 credit budget comfortably accommodated full single-pass coverage ($40 \text{ cycles} \times 7.5 = 300.0 \text{ credits}$), incurring $0.0$ cost overage and zero penalty.
   - In the Public phase, however, 160 events under a 220.0 credit budget meant that calling one metered tool per event cost $240.0$ credits ($9.09\%$ overage, costing $\approx 1.8$ points).
   - With another pass, we would implement adaptive budgeting via `ctx.tools.budget_remaining()`: dynamically skip lower-yield or expensive $2.0$-credit tools (`embedding_drift`, `feature_drift`) on alternating cycles when budget is constrained, while maintaining $1.0$-credit checks (`batch_profile`, `lineage_graph_slice`) that offer higher detection efficiency per credit.

2. **Stateful Change-Point and Cumulative Sum (CUSUM) Detectors:**
   - Single-event inspection is blind to subtle drifts that remain within $\pm 3\sigma$. Using `ctx.state` (which incurs zero RPC cost), we would implement online sequential test statistics such as CUSUM or Page-Hinkley tests on running feature means, batch row counts, and transform latencies.
   - By accumulating evidence across consecutive micro-batches, a stateful detector can detect sustained subtle shifts with high confidence while keeping the point-in-time false alarm rate at zero.
