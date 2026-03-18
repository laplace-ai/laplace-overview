# Staging Environment Proposal

## Problem

Today all backend services (DW, Loss Prediction, API) deploy directly to production. Frontend changes can be tested locally via `npm run dev`, but backend pipeline changes (new columns, schema changes, new service logic) can only be tested against the production database.

This works safely for **additive changes** (new columns, new tables) because old code ignores them. But it becomes risky for **destructive changes** (renaming columns, altering constraints, changing data flow logic) — a bad deploy could corrupt production data with no easy rollback.

## Proposed Architecture

Add a parallel dev environment using the same infrastructure, with separate Pub/Sub subscriptions and a separate database.

### Production (unchanged)

```
data-lake-updates (topic)
  └─ subscription: dw-prod-sub
       └─ cr-data-warehouse (Cloud Run, prod)
            └─ writes to: data_warehouse (Cloud SQL)
            └─ publishes to: data-warehouse-updates

data-warehouse-updates (topic)
  └─ subscription: dw-updates-loss-prediction-sub
       └─ cr-loss-prediction (Cloud Run, prod)
            └─ writes to: data_warehouse (Cloud SQL)

API Gateway → cr-platform-api (Cloud Run, prod)
  └─ reads from: data_warehouse (Cloud SQL)
```

### Dev environment (new)

```
data-lake-updates (topic)  ← same topic, new subscription
  └─ subscription: dw-dev-sub
       └─ cr-data-warehouse-dev (Cloud Run, dev)
            └─ writes to: data_warehouse_dev (Cloud SQL, same instance)
            └─ publishes to: data-warehouse-updates-dev (new topic)

data-warehouse-updates-dev (topic)
  └─ subscription: dw-updates-loss-prediction-dev-sub
       └─ cr-loss-prediction-dev (Cloud Run, dev)
            └─ writes to: data_warehouse_dev (Cloud SQL)

API Gateway (dev config) → cr-platform-api-dev (Cloud Run, dev)
  └─ reads from: data_warehouse_dev (Cloud SQL)
```

### Key design decisions

1. **Same Pub/Sub topic, separate subscriptions.** Each subscription gets its own copy of every message. Prod and dev process the same CSV files independently without interfering.

2. **Same Cloud SQL instance, separate database.** `CREATE DATABASE data_warehouse_dev` in the existing `dw-instance`. No second Cloud SQL instance needed. Cost is only the extra storage (~3GB).

3. **Separate Cloud Run services** (not revisions). `cr-data-warehouse-dev`, `cr-loss-prediction-dev`, `cr-platform-api-dev`. Each has its own env vars pointing to the dev database.

4. **Scale to zero when idle.** Dev services use `min_instances: 0`. Only cost when actively testing.

5. **Separate Pub/Sub topic for dev DW updates.** The dev DW publishes to `data-warehouse-updates-dev`, not the production topic. This prevents dev DW updates from triggering the production loss prediction service.

## Cost estimate

| Resource | Extra cost |
|----------|-----------|
| Cloud SQL storage | ~$1/month (3GB extra in same instance) |
| Cloud Run (3 dev services) | ~$0 when idle (scale to zero) |
| Pub/Sub (3 new subscriptions + 1 topic) | ~$0 (free tier covers it) |
| **Total** | **~$1-5/month** (only during active testing) |

## Setup steps

1. Create dev database:
   ```sql
   CREATE DATABASE data_warehouse_dev;
   ```

2. Create dev Pub/Sub resources:
   ```bash
   gcloud pubsub topics create data-warehouse-updates-dev
   gcloud pubsub subscriptions create dw-dev-sub --topic=data-lake-updates
   gcloud pubsub subscriptions create dw-updates-loss-prediction-dev-sub --topic=data-warehouse-updates-dev
   ```

3. Deploy dev Cloud Run services:
   ```bash
   # Each service deployed with --no-traffic and env vars pointing to dev database
   gcloud run deploy cr-data-warehouse-dev ...
   gcloud run deploy cr-loss-prediction-dev ...
   gcloud run deploy cr-platform-api-dev ...
   ```

4. Seed dev database with production data (one-time):
   ```bash
   pg_dump -t base_455_cleaned -t ctrc_loss_predictions data_warehouse | psql data_warehouse_dev
   ```

5. Point frontend `.env.local` to dev API for local testing.

## When to use

- **Additive changes** (new columns, new tables): test directly against prod (current workflow). Safe because old code ignores new schema.
- **Destructive changes** (rename columns, alter constraints, change data flow): test against dev environment first.
- **New service development** (new downstream consumers): develop against dev environment to avoid polluting production data.

## Priority

Low — current additive workflow is safe for now. Implement when we start doing schema migrations that modify existing columns or when we onboard a second tenant.
