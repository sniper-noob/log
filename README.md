
# 🚫 Remove `label`, `url`, and `timeBucketId` from Logging (Twitter, Reddit, YouTube)

Hi,

I propose removing `label`, `url`, and `timeBucketId` from logs across **Twitter**, **Reddit**, and **YouTube** to prevent miners from copying each other’s strategies.

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

---

### ✅ Recommendation

To protect originality and maintain fair competition, logs should **not include `label` and `url`** for Twitter, Reddit, or YouTube content.

Thanks.
