# Motor-Spirit-Gasoline-Blending-Optimization-Agent
A small Python tool for calculating and optimizing gasoline blends from a set of refinery component streams. It answers two questions:

1. **"If I mix these streams in these amounts, what RON and Sulfur do I get?"** → `calculate_blend()`
2. **"What flow rates do I need to hit a target RON?"** → `optimize_for_ron()`

---

## Requirements

- Python 3.8+
- `numpy`
- `pandas`
- `scipy`

Install dependencies:

```bash
pip install numpy pandas scipy
```

---

## Stream Data

The calculator works from a fixed table of available blending streams (`STREAM_DATA`), each with a maximum design flow, octane number (RON), blending strength (B-value), and sulfur content:

| Component | Max Flow (m³/hr) | RON | B-value | Sulfur (ppm) |
|---|---|---|---|---|
| Isomerate | 95.0 | 86.0 | 1.0 | 0.0 |
| Reformate | 80.0 | 98.0 | 1.0 | 0.0 |
| Prime G R/D (Cracked) | 105.0 | 86.8 | 1.4 | 10.0 |
| CCRU NHT DSN | 25.0 | 60.0 | 1.0 | 0.5 |
| OHCU Lt. Naphtha | 17.5 | 72.0 | 1.0 | 5.0 |
| Iso Octene (Polymers) | 8.5 | 115.0 | 2.6 | 45.0 |
| OHCU Hy. Naphtha | 2.5 | 60.0 | 1.0 | 8.0 |

To change the streams or their properties, edit the `STREAM_DATA` DataFrame at the top of the script.

---

## How the Math Works

**RON (octane) blending** uses a B-value-weighted formula rather than a simple average, since some streams (like cracked or polymer streams) don't blend linearly:

```
              Σ (RON_n × B_n × V_n)
Blend RON =  ------------------------
                 Σ (B_n × V_n)
```

where `V_n` is each stream's volume fraction of the total flow.

**Sulfur** blends as a straightforward volume-weighted average.

---

## Usage

### 1. Forward calculation — `calculate_blend(flows)`

Pass a dict of `{stream_name: flow_rate_m3hr}`. Any stream left out is treated as zero.

```python
from blend_calculator import calculate_blend

result = calculate_blend({
    "Isomerate": 60.0,
    "Reformate": 55.0,
    "Prime G R/D (Cracked)": 70.0,
    "CCRU NHT DSN": 15.0,
    "OHCU Lt. Naphtha": 10.0,
    "Iso Octene (Polymers)": 5.0,
    "OHCU Hy. Naphtha": 2.0,
})

print(result["blend_ron"])         # e.g. 89.7
print(result["blend_sulfur_ppm"])  # e.g. 4.2
print(result["total_flow"])        # e.g. 217.0
print(result["details"])           # per-stream breakdown (flow + vol fraction)
```

**Raises `BlendError` if:**
- Any flow is negative
- Any flow exceeds that stream's max design flow
- Total flow is zero

### 2. Reverse optimization — `optimize_for_ron(target_ron, total_throughput, max_sulfur_ppm=10.0)`

Given a target RON and a desired total throughput, finds a feasible flow rate for each stream that hits the target while respecting:
- Each stream's max design flow
- A maximum sulfur limit (default 10 ppm)

```python
from blend_calculator import optimize_for_ron

plan = optimize_for_ron(target_ron=91.0, total_throughput=250.0)

print(plan["achieved_ron"])          # ~91.0
print(plan["achieved_sulfur_ppm"])
print(plan["details"])               # recommended flow per stream
```

Among all flow combinations that satisfy the constraints, the solver favors the most **evenly spread** distribution across streams (not the only possible answer, just a sensible one).

**Raises `BlendError` if:**
- The requested throughput exceeds the combined max flow of all streams
- No feasible combination hits the target RON within the sulfur/flow limits

---

## Running the Demo

The script includes a `__main__` block with four worked examples: a forward blend calculation, an invalid-input error case, a successful RON optimization, and an unreachable RON target.

```bash
python blend_calculator.py
```

---
