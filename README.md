
# 🚫 Remove `label` and `url` from Logging (Twitter, Reddit, YouTube)

Hi,

I propose removing `label` and `url` from logs across **Twitter**, **Reddit**, and **YouTube** to prevent miners from copying each other’s strategies.

Currently, logs expose:

- **label** 
- **url**
- **timeBucketId**
- **bytes size**

This reveals **what** was mined, **when**, and **from where** — making it easy for others to duplicate high-value buckets. I've personally seen my byte count drop sharply after validation, suggesting others copied my submitted labels and URLs after viewing them in logs.

---

### ⚖️ Scoring Impact – With Example

The scoring logic in `read_miner_index()` rewards miners using this formula:

> **scorable_bytes = (contentSizeBytes × contentSizeBytes) / totalBucketBytes**

Suppose 3 miners submit data for the same bucket:

| Miner   | contentSizeBytes | totalBucketBytes | Scorable Bytes                 |
| ------- | ---------------- | ---------------- | ------------------------------ |
| Miner A | 10,000           | 30,000           | (10,000² / 30,000) = **3,333** |
| Miner B | 9,000            | 30,000           | (9,000² / 30,000) = **2,700**  |
| Miner C | 5,000            | 30,000           | (5,000² / 30,000) = **833**    |

As you can see, **just 1,000 more bytes** gives Miner A a significantly higher score — despite only contributing ~33% of total data.

But if others see which `label`, `url`, `bytes size`, and `timeBucketId` performed well, they can easily scrape the same or more and **overtake the original miner** in the next round.


# 📊 Miner Score Impact from Running the Same Bucket where Miner A running 6 Different Hotkeys

This table shows the impact when **Miner A runs 6 separate miners**, each submitting the same 10,000 bytes for the same `(label, timeBucketId)`. 

The result: Miner A gains double the total score, while other miners are heavily penalized due to inflated `totalBucketBytes`.

| Miner         | contentSizeBytes | totalBucketBytes | Scorable Bytes                  | % Change vs Original |
|---------------|------------------|------------------|----------------------------------|-----------------------|
| Miner A (1)   | 10,000           | 90,000           | (10,000² / 90,000) = **1,111**   | **−67%** (was 3,333)  |
| Miner A (2)   | 10,000           | 90,000           | **1,111**                        | —                     |
| Miner A (3)   | 10,000           | 90,000           | **1,111**                        | —                     |
| Miner A (4)   | 10,000           | 90,000           | **1,111**                        | —                     |
| Miner A (5)   | 10,000           | 90,000           | **1,111**                        | —                     |
| Miner A (6)   | 10,000           | 90,000           | **1,111**                        | —                     |
| **Total A**   | 60,000           | 90,000           | **6,666**                        | **+100%**             |
| Miner B       | 9,000            | 90,000           | (9,000² / 90,000) = **900**      | **−67%** (was 2,700)  |
| Miner C       | 5,000            | 90,000           | (5,000² / 90,000) = **278**      | **−67%** (was 833)    |

---

### 🔍 Summary
- Miner A **doubles its total score** by duplicating effort across 6 nodes.
- Miner B and C suffer ~67% score loss without changing their contribution.
- This illustrates how self-replication distorts fairness in the scoring algorithm.


---

### ✅ Recommendation

To protect originality and maintain fair competition, logs should **not include `label` and `url`** for Twitter, Reddit, and YouTube content.

Thanks.
