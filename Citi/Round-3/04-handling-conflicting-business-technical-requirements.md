# Handling Conflicting Requirements Between Business & Technical Teams

## Question
How do you handle situations where business stakeholders demand real-time streaming data with strict deadlines, but technical/architectural constraints require batch processing or data governance validation?

---

## 1. My 4-Step Resolution Strategy

```
[ 1. Understand Core Business Objective & True SLA Needs ]
                       │
                       ▼
[ 2. Quantify Technical Trade-offs (Cost, Complexity, Maintenance) ]
                       │
                       ▼
[ 3. Propose a Pragmatic Hybrid Compromise (e.g. 15m Micro-Batch) ]
                       │
                       ▼
[ 4. Align on Phased Delivery & Document SLA Agreements ]
```

---

## 2. Real-World Scenario
- **Conflict:** The front-office trading desk requested "sub-second real-time streaming" for all historical risk reports, demanding delivery within 2 weeks.
- **Technical Reality:** True sub-second streaming for petabyte historical risk models would multiply cloud infrastructure costs by 5x and required rewriting 40+ complex batch SQL models.
- **Resolution:**
  1. Facilitated a discovery session to uncover that traders only needed real-time updates for **today's active intraday positions**, while historical comparisons were reviewed only once per morning.
  2. Implemented a **hybrid solution**: Sub-second streaming for intraday position alerts using Spark Structured Streaming, paired with an optimized 15-minute micro-batch for standard risk curves.
  3. Delivered on time within budget, satisfying 100% of the desk's operational requirements.
