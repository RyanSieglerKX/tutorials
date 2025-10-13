# 🚀 ai-libs Tutorials

This folder contains concise, step-by-step tutorials designed to help developers quickly understand how to integrate KDB-X with AI applications.

**Important!** Intel-based macOS is not supported at this time.

## 📖 Tutorials
1️⃣ Temporal Similarity Search (TSS) on NYSE trades
- Learn how to apply Temporal Similarity Search (TSS) on NYSE trade data to find patterns and trends in time series data.
- Conducting TSS with efficient by clauses to search for patterns like spikes, dips, or repeating trends.
- Analyzing multiple symbols and their time series patterns.

2️⃣ HNSW with Hacker News Embeddings
- Creating a historical database (HDB) with Hacker News data and embeddings
- Building an HNSW index per partition for fast nearest neighbor searches
- Querying the database using both q and Python

3️⃣ TSS on Partitioned Bitcoin Data
- Apply Temporal Similarity Search (TSS) across a partitioned database filled with Bitcoin data.
- Efficient TSS query execution across partitions.
- Handling temporal pattern matches that span partition boundaries.

4️⃣ IVF with Wikipedia Embeddings 
- Learn how to use Inverted File Index (IVF) search on partitioned databases containing Wikipedia embeddings.
- Partitioning data using K-Means clustering to improve search efficiency.
- Performing IVF searches and leveraging the ai-libs' flat functionality.

5️⃣ Combining Structured and Unstructured Data
-  Discover how to combine structured (Yahoo finance feed) and unstructured (SENS announcments) data
- Fetching Yahoo Finance ticker data using PyKX and creating embeddings with PyTorch on SENS data
- Integrating time series analysis with TSS and nearest neighbor searches with HNSW 

6️⃣ Fuzzy Filtering
- This tutorial walks through two examples of where it may be useful to introduce Fuzzy Filters, such as for queries or data that may contain typos, international spelling variations, or misspellings, and would normally result in no relevant search results, or incorrect data being retrieved.

7️⃣ DTW Getting Started
- Discover how to use Dynamic Time Warping on financial NYSE data to identify similar patterns in price movements, even when those patterns occur at different speeds or times.

8️⃣ BM25 & Hybrid Search
- Explore how to combine traditional BM25 ranking with vector-based similarity for powerful hybrid search capabilities.
- Learn how to blend both approaches for optimal retrieval performance using q and Python.
  
## 🤝 Got a question?
Want to connect with other developers or get help? Join our Slack community https://kx.com/slack or ask a question on https://forum.kx.com
