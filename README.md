# SupplierReliabilityScore
 This score evaluates wallets based only on their behavior as suppliers of assets to the protocol. Higher scores reflect patterns indicative of stable, potentially human-driven supply activity. Lower scores reflect minimal engagement, or patterns potentially associated with bots.
he raw JSON data from the selected files was processed using Python and pandas:

->The data under the primary "deposits" key in each file was loaded.
Nested JSON structures were flattened using pd.json_normalize for easier handling.

->Key fields were extracted: wallet (account.id), amount, symbol (asset.symbol), timestamp, tx_hash (hash), and token_address (asset.id).

->The event type was standardized to Supply.

->Data types were corrected (timestamp to datetime, amount to numeric). Rows with missing essential information after conversion were dropped.

->The combined dataset was sorted by timestamp.
To score wallets, we needed to summarize their deposit history into meaningful features:

Method: The transaction data was grouped by wallet address. Various metrics were calculated for each wallet using aggregations.

Key Features Engineered & Why:

->supply_count: How many times did they deposit? (Basic activity measure).

->supplier_activity_duration_days: How long between their first and last deposit? (Measures longevity/engagement).

->avg_time_between_supplies_days: On average, how often did they deposit during their active period? (Measures frequency).

->supplies_per_day: How many deposits per day of activity? (Measures intensity, potentially flags bots if very high).

->days_since_last_supply: How recently was their last deposit? (Measures recency relative to the data's end date).

->distinct_supplied_assets: How many different types of tokens did they supply? (Measures diversity of engagement).

->cv_supply_amount_raw: How consistent were their deposit amounts (Coefficient of Variation: Std Dev / Mean)? (Measures behavioral consistency, low CV = more consistent amounts, useful across different tokens).
->Approach: Since we had no labels defining "good" or "bad" suppliers, an unsupervised clustering approach was needed. K-Means was chosen because it's effective at partitioning data into groups based on feature similarity and provides interpretable cluster centers (centroids).

->Preparation: The engineered features were scaled using StandardScaler. This was essential because K-Means relies on Euclidean distance, and features had vastly different ranges and variances (as seen in visualizations), requiring normalization.

->Choosing K (Number of Clusters):

    ->The Elbow Method (plotting inertia vs. K) was used.

    ->Silhouette Scores (measuring cluster cohesion and separation) were also calculated for different K values.

->Decision: K=4. The Elbow plot showed a reasonably clear bend at K=4. While Silhouette scores (~0.37 for K=4, slightly increasing for higher K) indicated imperfect separation (common in real-world behavioral data), K=4 produced four distinct and highly interpretable behavioral profiles based on the cluster centroids. Prioritizing interpretability for the scoring logic, K=4 was selected.
A custom, multi-step process translated the cluster assignments into the final score:

->Step 1: Interpreting & Ranking Clusters: The centroids (average feature values) for each of the 4 clusters were analyzed to understand the typical behavior they represented:

    I)Cluster 0: Longest average duration, highest diversity, recent -> "Established, Diverse Suppliers".

    II)Cluster 1: Minimal activity/duration, single asset, oldest  "Single-Deposit / Short-Term Suppliers".

    III)Cluster 2: Short duration but high intensity -> "Short-Burst, Active Suppliers".

    IV)Cluster 3: Single wallet, extreme frequency/count -> "Hyper-Active Single Supplier (Outlier)".

    Based on the goal of identifying reliable suppliers, these were ranked (0=Worst, 3=Best): Rank 0: C1, Rank 1: C3, Rank 2: C2, Rank 3: C0.

->Step 2: Assigning Base Score Ranges: The 0-100 score range was divided equally among the ranks. Wallets in Rank 0 clusters get a base score between 0-25, Rank 1 between 25-50, Rank 2 between 50-75, and Rank 3 between 75-100.

->Step 3: Refining Scores Within Clusters: To make the score non-trivial and reflect finer differences, a refinement was applied within each cluster's base range:

    I)Refinement Feature: supplier_activity_duration_days was chosen. Rationale: Even among wallets with similar overall profiles (same cluster), longer activity suggests more sustained engagement.

    II)Method: For each wallet, its duration was scaled to a 0-1 value based on the minimum and maximum duration found only among wallets in its specific cluster.
    
    III)Final Score: This 0-1 scaled duration was used to linearly interpolate the final score within the cluster's base score range (e.g., a scaled duration of 0 gets the cluster's min base score, 1 gets the max base score). This ensures wallets with longer duration relative to their peers in the same cluster receive a higher score within that cluster's assigned range.

