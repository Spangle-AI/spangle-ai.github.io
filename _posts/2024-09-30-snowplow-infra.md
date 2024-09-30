---
layout: post
title: Deploying Snowplow 2 - Infrastructure
---

This is part 2 of our post that explains how to deploy Snowplow GCP and GKE. In [part 1]({% post_url 2024-09-28-snowplow-architecture %}) we gave a high level overview of the architecture. In this part we will explain how to use Terraform to create the required infrastructure in GCP. You probably want to add the following defintions in a folder called _snowplow_ insides _modules_ folder and then import it from main.

We go over these resources:

1. [Variables](#variables)
2. [Google Cloud Storage](#google-cloud-storage)
3. [PostgreSQL](#postgresql)
4. [BigQuery Dataset](#bigquery-dataset)
5. [Networking and IP Address](#networking-and-ip-address)
6. [PubSub topics and subscriptions](#pubsub-topics-and-subscriptions)
7. [PubSub to GCS Permissions](#pubsub-to-gcs-permissions)
8. [Service Account and Permissions](#service-account-and-permissions)

## Variables

Let's start with the set of variables that our infrastructure is going to need:

```tf
# modules/snowplow/variables.tf

variable "project_id" {
  description = "The ID of the GCP project"
  type        = string
}

variable "region" {
  description = "The region for GCP resources"
  type        = string
}

variable "igludb_instance_id" {
  description = "The ID of the Cloud SQL PostgreSQL instance for Iglu"
  type        = string
}

variable "igludb_db_name" {
  description = "Database name for Iglu"
  type        = string
}

variable "igludb_username" {
  description = "The username for the Iglu database"
  type        = string
}

variable "igludb_password" {
  description = "The password for the Iglu database user"
  type        = string
  sensitive   = true # Mark as sensitive to protect the password
}
```

## Google Cloud Storage

Next we need three storage buckets for bad rows, failed inserts, etc:

```tf
# modules/snowplow/storage.tf

resource "google_storage_bucket" "bq_loader_dead_letter_bucket" {
  name                        = "spangle-snowplow-bq-loader-dead-letter"
  location                    = var.region
  storage_class               = "STANDARD"
  force_destroy               = true
  uniform_bucket_level_access = true

  versioning {
    enabled = false
  }
}

resource "google_storage_bucket" "bad_1_bucket" {
  name                        = "spangle-snowplow-bad-1"
  location                    = var.region
  storage_class               = "STANDARD"
  force_destroy               = true
  uniform_bucket_level_access = true

  versioning {
    enabled = false
  }
}

resource "google_storage_bucket" "bq_bad_rows_bucket" {
  name                        = "spangle-snowplow-bq-bad-rows"
  location                    = var.region
  storage_class               = "STANDARD"
  force_destroy               = true
  uniform_bucket_level_access = true

  versioning {
    enabled = false
  }
}
```

## PostgreSQL

In case you already have a cloudsql PostgreSQL setup you can skip next step, otherwise we need to create a cloudsql PostgreSQL database. Here is just an example setup:

```tf
# modules/snowplow/sql_database.tf

resource "google_sql_database_instance" "dev_postgres_instance" {
  name             = "dev-postgres"
  database_version = "POSTGRES_16"
  region           = var.region


  settings {
    tier                        = "db-custom-2-8192"
    deletion_protection_enabled = true

    backup_configuration {
      enabled                        = true
      location                       = "us"
      point_in_time_recovery_enabled = true
    }

    maintenance_window {
      update_track = "canary"
    }
  }
}
```

\
Then we need to create a database for the Iglu server on the PostgreSQL database created above:

```tf
# modules/snowplow/sql_database.tf

# Create a database in the existing Cloud SQL PostgreSQL instance
resource "google_sql_database" "iglu_db" {
  name     = var.igludb_db_name     # Replace with your desired database name
  instance = var.igludb_instance_id # Reference the existing Cloud SQL instance
  project  = var.project_id         # Reference the project ID
}

# Create a user for the database with a password
resource "google_sql_user" "iglu_postgres_user" {
  name     = var.igludb_username    # Use the username variable
  password = var.igludb_password    # Use the password variable
  instance = var.igludb_instance_id # Reference the existing Cloud SQL instance
  project  = var.project_id         # Reference the project ID
}
```

If you are creating the database with the commands above, make sure you add a dependency declaration to the database definitions. Something like:

```tf
  depends_on = [
    google_sql_database_instance.dev_postgres_instance
  ]
```

to both `google_sql_database.iglu_db` and `google_sql_database.iglu_postgres_user` above.

## BigQuery Dataset

Next, let's create the BigQuery dataset. We call the dataset `snowplow`, you can of course change that to something else, but you need to make sure you update the service configuration files, that we will go over in part 3, accordingly.

```tf
# modules/snowplow/bigquery.tf

# Create a BigQuery dataset with the specified location and description
resource "google_bigquery_dataset" "snowplow_event_dataset" {
  dataset_id  = "snowplow"
  description = "Snowplow event dataset"
  location    = "US"

  project = var.project_id
}
```

## Networking and IP Address

Next, let's create the static IP address that is used as the tracker endpoint:

```tf
# modules/snowplow/network.tf

# Create a global static IP address
resource "google_compute_global_address" "snowplow_ingress_ip" {
  name       = "snowplow-ingress-ip"
  ip_version = "IPV4"
}
```

## PubSub topics and subscriptions

We also need to create the six PubSub topics and their related subscriptions that we mentioned in Part 1.

```tf
# modules/snowplow/pubsub.tf

locals {
  pubsub_topics = [
    # Tuple format: ("topic_name", subscription_needed)
    ["snowplow-bad-1", false],
    ["snowplow-bq-bad-rows", false],
    ["snowplow-bq-loader-server-failed-inserts", true],
    ["snowplow-bq-loader-server-types", true],
    ["snowplow-enriched", true],
    ["snowplow-raw", true]
  ]
}

resource "google_pubsub_topic" "pubsub_topics" {
  for_each = { for topic in local.pubsub_topics : topic[0] => topic }
  name     = each.value[0]
}

resource "google_pubsub_subscription" "pubsub_subscriptions" {
  for_each = { for topic in local.pubsub_topics : topic[0] => topic if topic[1] }
  name     = each.value[0]
  topic    = each.value[0]

  expiration_policy {
    ttl = "604800s" # Set TTL to 7 days (604800 seconds)
  }
}

resource "google_pubsub_subscription" "bad_1_subscription" {
  name  = "snowplow-bad-1-gcs"
  topic = google_pubsub_topic.pubsub_topics["snowplow-bad-1"].id

  cloud_storage_config {
    bucket = google_storage_bucket.bad_1_bucket.name

    filename_prefix          = ""
    filename_suffix          = "-${var.region}"
    filename_datetime_format = "YYYY-MM-DD/hh_mm_ssZ"

    # max_bytes    = 1000000
    # max_duration = "300s"
    # max_messages = 1000
  }

  depends_on = [
    google_storage_bucket.bad_1_bucket,
    google_storage_bucket_iam_member.admin,
  ]
}

resource "google_pubsub_subscription" "bq_bad_rows_subscription" {
  name  = "snowplow-bq-bad-rows-gcs"
  topic = google_pubsub_topic.pubsub_topics["snowplow-bq-bad-rows"].id

  cloud_storage_config {
    bucket = google_storage_bucket.bq_bad_rows_bucket.name

    filename_prefix          = ""
    filename_suffix          = "-${var.region}"
    filename_datetime_format = "YYYY-MM-DD/hh_mm_ssZ"

    # max_bytes    = 1000000
    # max_duration = "300s"
    # max_messages = 1000
  }

  depends_on = [
    google_storage_bucket.bq_bad_rows_bucket,
    google_storage_bucket_iam_member.admin,
  ]
}
```

## PubSub to GCP Permissions

Somewhat confusingly, we should explicitly allow PubSub to GCS service account to write to the GCS buckets we mentioned above. In other words:

```tf
# modules/snowplow/iam.tf

data "google_project" "project" {
}

locals {
  bucket_names = [
    google_storage_bucket.bad_1_bucket.name,
    google_storage_bucket.bq_bad_rows_bucket.name,
  ]
}

resource "google_storage_bucket_iam_member" "admin" {
  for_each = toset(local.bucket_names)

  bucket = each.value
  role   = "roles/storage.admin"
  member = "serviceAccount:service-${data.google_project.project.number}@gcp-sa-pubsub.iam.gserviceaccount.com"
}
```

## Service Account and Permissions

Next we create service account that is going to be used by all the Snowplow services and we give it all the permissions that these services need. Note that, here we are using one shared service account just for simplicity. Ideally, it's better to create different service accounts for different services and give them exactly the minimum permissions that they need.

Problably the most non-trivial part of this declaration is the last part, **workload_identity_user_binding** where we allow GKE service account `snowplow/snowplow-service-account` i.e. service account `snowplow-service-account` in Kubernetes namespace `snowplow` to bind to `snowplow` service account in GCP. In other words, we give our kubernetes services the ability to use GKE service account `snowplow/snowplow-service-account` which can then bind to `snowplow` service account we create below on GCP. More on that in Part 3.

```tf
# modules/snowplow/service_accounts.tf

# Combined Service Account: snowplow
resource "google_service_account" "snowplow" {
  account_id   = "snowplow"
  display_name = "snowplow"
  description  = "Combined service account with all permissions for Snowplow"
}

# List of roles to be assigned to the service account
locals {
  roles = [
    "roles/bigquery.dataEditor",
    "roles/logging.logWriter",
    "roles/pubsub.publisher",
    "roles/pubsub.subscriber",
    "roles/pubsub.viewer",
    "roles/storage.objectViewer"
  ]
}

# Assign all roles to the service account in a loop
resource "google_project_iam_member" "combined_iam_roles" {
  for_each = toset(local.roles)
  project  = var.project_id
  role     = each.value
  member   = "serviceAccount:${google_service_account.snowplow.email}"
}

# Add IAM policy binding to the Google Cloud Storage bucket
resource "google_storage_bucket_iam_member" "bq_loader_dead_letter_bucket_binding" {
  bucket = google_storage_bucket.bq_loader_dead_letter_bucket.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.snowplow.email}"
}

# Add Workload Identity User binding
resource "google_service_account_iam_binding" "workload_identity_user_binding" {
  service_account_id = google_service_account.snowplow.name

  role = "roles/iam.workloadIdentityUser"

  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[snowplow/snowplow-service-account]"
  ]
}
```

Nice! by now we should have eight files in `modules/snowplow` folder. Feel tree to `terraform plan` and `terraform apply`.

```
    modules/snowplow
    ├── README.md
    ├── bigquery.tf
    ├── iam.tf
    ├── network.tf
    ├── pubsub.tf
    ├── service_accounts.tf
    ├── sql_database.tf
    ├── storage.tf
    └── variables.tf
```

In [Part 3]({% post_url 2024-09-30-snowplow-cluster %}) we will go over the Kubernetes cluster deployment.
