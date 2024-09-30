---
layout: post
title: Deploying Snowplow 3 - Kubernetes Cluster
---

This is the Part 3 of our post that explains how to deploy Snowplow on GCP and GKE. In [part 1]({% post_url 2024-09-28-snowplow-architecture %}) we gave a high level overview of the architure of our deployment. In [part 2]({% post_url 2024-09-30-snowplow-infra %}) we went into full details of how to create the infrastrucure required for this deployment using Terraform. In this part, we explain in details, how to deploy the services on Kubernetes. Note that the implemtation mentioned here does not include horizontal scaling HPA, and must be manually updated to handle increased traffic.

We will go over HPA in a future post. In this post, we go over these resources:

1. [Namespace](#namespace)
2. [Service Account](#service-account)
3. [Iglu Service](#iglu-service)

## Namespace

In our implementation, we deploy everything in `snowplow` namespace. We need to create the namespace first:

```yaml
# snowplow/namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: snowplow
```

## Service Account

As we mentioned in part 2, in order to authenticate our GKE deployments, we use GKE's [Workload Identity Federation](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity). The way this works is that we create a GKE service account named `snowplow-service-account` in `snowplow` namespace and let our deployments to use this service account by mentioning:

```
serviceAccountName: snowplow-service-account
```

in deployment's template spec. In part 2, we allowed this service account to bind to GCP service account `snowplow` that we created specially for this purpose and gave it all the required permissions. So let's create the GKE service account:

```yaml
# snowplow/service-account.yaml

apiVersion: v1
kind: ServiceAccount

metadata:
  annotations:
    iam.gke.io/gcp-service-account: snowplow@<GCP-PROJECT>.iam.gserviceaccount.com
  name: snowplow-service-account
  namespace: snowplow
```

Note that `snowplow@<GCP-PROJECT>.iam.gserviceaccount.com` is the name of GCP service account we created in part 2. You should change `<GCP-PROJECT>` to the name of your GCP project.

## Iglu Service

Perfect. Next, let's deploy Iglu service. [Iglu Server](https://docs.snowplow.io/docs/pipeline-components-and-applications/iglu/iglu-repositories/iglu-server/) in Snowplow's documentation words:

> The Iglu Server is an Iglu schema registry which allows you to publish, test and serve schemas via an easy-to-use RESTful interface.

We divide Iglu service declaration (and all other service to come next) into a config file and a deployment file. Here is the config file for the Iglu service. Here you can find [full documentation for Iglu Server Configuration](https://docs.snowplow.io/docs/pipeline-components-and-applications/iglu/iglu-repositories/iglu-server/reference/). Note that this is the documentation for the latest version, but it gives a very good overview of the options for older version.

Also note that we make sure we use an older image that does not require us to accept the new [Snowplow Limited Use License](https://docs.snowplow.io/limited-use-license-1.0/). Therefore if you don't want to be limited by the new licene, make sure you use an older version and **make sure you don't have**:

```hocon
license {
  accept = true
}
```

in your config. This applies to all the configurations we show next as well.

```yaml
# snowplow/iglu-config.yaml

kind: ConfigMap
apiVersion: v1
metadata:
  name: iglu-configmap
  namespace: snowplow

data:
  iglu-server.hocon: |
    {
      "repoServer": {
        "interface": "0.0.0.0"
        "port": 8080
        "threadPool": "cached"
        "maxConnections": 2048
      }
      "database": {
        "type": "postgres"
        "host": "<igludb_ip_address>"
        "port": 5432
        "dbname": "<igludb_db_name>"
        "username": "<igludb_username>"
        "password": "<igludb_password>"
        "driver": "org.postgresql.Driver"
        pool: {
          "type": "hikari"
          "maximumPoolSize": 5
          connectionPool: {
            "type": "fixed"
            "size": 4
          }
          "transactionPool": "cached"
        }
      }
      "debug": false
      "patchesAllowed": true
      "superApiKey": "<UUID>"
    }
```

Note that you need to make sure you full the following values to match with the ones we used in part 2 to create the resources:

- igludb_ip_address
- igludb_db_name
- igludb_username
- igludb_password

Also, you need to generate a UUID for `superApiKey`. Make sure to keep it private. After you created and applied the config file, let's create and apply the deployment:

```yaml
# snowplow/iglu.yaml

apiVersion: apps/v1
kind: Deployment

metadata:
  name: iglu-server
  namespace: snowplow

spec:
  selector:
    matchLabels:
      app: iglu

  replicas: 2
  revisionHistoryLimit: 1

  template:
    metadata:
      labels:
        app: iglu

    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000

      containers:
        - name: iglu-server
          image: snowplow/iglu-server:0.11.0
          command:
            [
              "/home/snowplow/bin/iglu-server",
              "--config",
              "/snowplow/config/iglu-server.hocon",
            ]
          imagePullPolicy: "IfNotPresent"

          env:
            - name: JAVA_OPTS
              value: -Dorg.slf4j.simpleLogger.defaultLogLevel=info

          volumeMounts:
            - name: iglu-config-volume
              mountPath: /snowplow/config

          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "1.5Gi"

      volumes:
        - name: iglu-config-volume
          configMap:
            name: iglu-configmap
            items:
              - key: iglu-server.hocon
                path: iglu-server.hocon

---
apiVersion: v1
kind: Service

metadata:
  name: iglu-server-service
  namespace: snowplow

spec:
  selector:
    app: iglu

  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```
