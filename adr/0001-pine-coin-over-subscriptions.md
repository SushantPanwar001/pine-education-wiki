# Pine Coin virtual currency over subscription model

Status: accepted

The platform uses a virtual currency (Pine Coin) for assessment purchases instead of subscription tiers. 1 INR = 5 Pine Coins. Users purchase coin bundles via admin credit (now) or payment gateway (future). Each exam has a configurable price in coins, deducted atomically when starting an assessment. No refunds. This model was chosen over subscriptions because it offers finer-grained monetization, lower commitment for students, and simpler implementation while awaiting App Store account verification. The `subscriptionTier` field on the user table is vestigial and should be removed.
