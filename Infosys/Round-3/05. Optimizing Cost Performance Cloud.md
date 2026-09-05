# Case Study: Optimizing Cost and Performance in Cloud Data Solutions

## Question
Describe a real-world project where you analyzed cloud consumption and significantly reduced compute/storage costs or improved pipeline performance.

---

## 1. Situation
Our team was managing an Azure Databricks analytics platform running 24/7 interactive clusters for scheduled batch ETL workflows, resulting in monthly cloud bills exceeding \$45,000.

---

## 2. Optimization Initiatives Implemented
1. **Migrated from Interactive to Automated Job Clusters:** Converted all scheduled ADF notebooks to run on ephemeral **Single-Job Clusters**, saving 40% on Databricks Unit (DBU) pricing.
2. **Utilized Azure Spot VMs for Worker Nodes:** Configured worker nodes to run on **Spot Instances** with auto-scaling, reducing VM compute costs by 65%.
3. **Automated Delta Lake Vacuuming & Storage Lifecycle:** Scheduled weekly `VACUUM` jobs to prune tombstoned files and moved Bronze Parquet data older than 90 days to ADLS Gen2 Cool/Archive tiers.

---

## 3. Financial & Performance Impact
- **Cost Reduction:** Reduced overall monthly Azure infrastructure spend from **\$45,000 to \$18,500** (a 58% ongoing cost reduction).
- **Performance:** Dynamic autoscaling improved morning peak batch processing times by 25%.
