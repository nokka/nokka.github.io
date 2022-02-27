---
title: "Cloud SQL"
date: "2021-11-14"
tags: ["cloudsql"]
draft: false
---

1. Make sure billing is enabled
2. Add Cloud SQL admin role
3. Enable the CloudSQL API
4. Go to MYSQL -> Create instance -> MYSQL
5. Create a user

```go
package main

func main() {
	// Context used for mongo operations, to time them out and cancel their context.
	mgoCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err != nil {
		log.Println("failed to create context with timeout for mongodb connection", err)
		os.Exit(0)
	}

	client, err := mongo.Connect(mgoCtx, clientOptions)
	if err != nil {
		log.Println("failed to connect to mongodb", err)
		os.Exit(0)
	}

	err = client.Ping(mgoCtx, readpref.Primary())
	if err != nil {
		log.Println("failed to ping mongodb", err)
		os.Exit(0)
	}

	log.Println("connected to mongodb")

	// Repositories.
	characterRepository := mgo.NewCharacterRepository(databaseName, client)
	statisticsRepository := mgo.NewStatisticsRepository(databaseName, client)

	// Business logic services.
	parser := parsing.NewParser(d2sPath)
	characterService := character.NewService(parser, characterRepository, cd)
	statisticsService := statistics.NewService(statisticsRepository)

	// Channel to receive errors on.
	errorChannel := make(chan error)

	// Credentials for posting statistics map.
	credentials := map[string]string{
		statisticsUser: statisticsPassword,
	}
	// HTTP server.
	go func() {
		httpServer := httpserver.NewServer(
			httpAddress,
			characterService,
			statisticsService,
			credentials,
			cors,
			logging,
		)
		errorChannel <- httpServer.Open()
	}()

	// Capture interupts.
	go func() {
		c := make(chan os.Signal, 1)
		signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
		errorChannel <- fmt.Errorf("got signal %s", <-c)
	}()

	// Listen for errors indefinitely.
	if err := <-errorChannel; err != nil {
		os.Exit(1)
	}
}
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

