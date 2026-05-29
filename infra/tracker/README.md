# Tracker

A Cloud Run service that receives bizevent payloads from challenge Codespaces and forwards them to a Dynatrace tenant.

It validates that every incoming event has `type: offon-challenges`, a known `action`, and all required fields (`adventure`, `level`, `session.id`) before ingesting. Anything else is rejected with a 400.

## Deployment

Deployed manually via the Google Cloud CLI. One-time secret setup:

```sh
echo -n "https://your-tenant.live.dynatrace.com" | gcloud secrets create dt-tenant-url --data-file=-
echo -n "dt0c01.xxx" | gcloud secrets create dt-api-token --data-file=-

PROJECT_NUMBER=$(gcloud projects describe $(gcloud config get-value project) --format='value(projectNumber)')
gcloud secrets add-iam-policy-binding dt-tenant-url \
  --member="serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
gcloud secrets add-iam-policy-binding dt-api-token \
  --member="serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

Deploy:

```sh
gcloud run deploy tracker \
  --source infra/tracker \
  --region europe-west1 \
  --allow-unauthenticated \
  --set-secrets DT_TENANT_URL=dt-tenant-url:latest,DT_API_TOKEN=dt-api-token:latest
```

To update the service, re-run the deploy command.
