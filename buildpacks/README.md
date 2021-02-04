# Cloud Run with [Google Cloud Buildpacks](https://github.com/GoogleCloudPlatform/buildpacks)

1. Configurations

```bash
export PROJECT_ID="$( gcloud config get-value project )"
gcloud config set run/platform managed
gcloud config set run/region asia-northeast1
```

2. Let's deploy :)

```bash
# Build & Run
gcloud builds submit --pack image="gcr.io/${PROJECT_ID}/my-app"
gcloud run deploy --image "gcr.io/${PROJECT_ID}/my-app"

# Replace the service
gcloud beta run services replace service.yaml

# Build & push the next version
gcloud builds submit --pack image="gcr.io/${PROJECT_ID}/my-app:green"
gcloud beta run deploy --image gcr.io/${PROJECT_ID}/my-app:green --no-traffic

# Granual deployment
gcloud run services update-traffic my-app --to-revisions green=50
gcloud run services describe my-app

# Rollback
previous_version=
gcloud run services update-traffic my-app --to-revisions "${previous_version}"
```
