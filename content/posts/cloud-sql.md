---
title: "Cloud SQL"
date: "2021-11-14"
tags: ["cloudsql"]
draft: false
---

## CloudSQL

1. Make sure billing is enabled
2. Add Cloud SQL admin role
3. Enable the CloudSQL API
4. Go to MYSQL -> Create instance -> MYSQL
5. Create a user

```


### Connecting from a GKE cluster
The Cloud SQL Auth proxy recommended even for private IPs

Create secret with database credentials
Get the connection name lendo-platform-sandbox-300921:europe-north1:platform-sandbox-mysql

```
gcloud sql instances describe platform-sandbox-mysql | grep -i "connectionName"
```

Create service account to connect to the cloudSQL instance
Create a credentials.json and create a secret from it
Mount the secret into the sidecar pod

```
kubectl create secret generic cloud-proxy-sa-secret --from-file=proxy-sa.json
```

Port forward to the mysql container
kat port-forward cloud-sql-proxy-56bd9db96c-hx2v8 3306:3306

mysql -u root -h 127.0.0.1 -p


## Resources
https://cloud.google.com/sql/docs/mysql/quickstart
https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine

