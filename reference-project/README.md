# CU05 Quarkus cloud native reference project

_You are now on tag **6-kubernetes**_

This project uses Quarkus, the Supersonic Subatomic Java Framework.

If you want to learn more about Quarkus, please visit its website: https://quarkus.io/ .

## Database & Jaeger & Prometheus

The app uses data persistence and you need a working database to use it. We also use tracing and metrics collector.
There is a `docker-compose.yaml` in the root of this repo that provides all of them.
You can start the DB and Jaeger containers using simple cmd:
```
docker-compose up
```
If you want to use other DB, modify the params in `application.properties`

To access Jaeger UI(tracing): http://localhost:16686  
To access Prometheus(metrics): http://localhost:9090/graph
To access health check of our app: http://localhost:8080/q/health

## Running the application in dev mode

You can run your application in dev mode that enables live coding using:
```shell script
./mvnw compile quarkus:dev
```

> **_NOTE:_**  Quarkus now ships with a Dev UI, which is available in dev mode only at http://localhost:8080/q/dev/.

## Running tests

You can run tests from your IDE or via maven. Simply run `./mvnw test ` or `./mvnw package`

## Tkit quarkus

Adding tkit libs to our project gives us several new features. Check the logs for example, all our business methods are now logged and timed. 
The REST API now handles exceptions gracefully(as JSON response), and we get server side pagination with very little effort. 
Our tests are now real integration tests with real postgres DB, and are stuitable for CI envs.

## Access your REST endpoint

Go to http://localhost:8080/animals


## OpenAPI & Swagger UI

With your app running, go to http://localhost:8080/q/swagger-ui to see the Swagger UI visualizing your API. You can access the YAML OpenAPI schema under http://localhost:8080/q/openapi

## Packaging and running the application

The application can be packaged using:
```shell script
./mvnw package
```
It produces the `quarkus-run.jar` file in the `target/quarkus-app/` directory.
Be aware that it’s not an _über-jar_ as the dependencies are copied into the `target/quarkus-app/lib/` directory.

If you want to build an _über-jar_, execute the following command:
```shell script
./mvnw package -Dquarkus.package.type=uber-jar
```

The application is now runnable using `java -jar target/quarkus-app/quarkus-run.jar`.

## Creating a native executable

You can create a native executable using: 
```shell script
./mvnw package -Pnative
```

Or, if you don't have GraalVM installed, you can run the native executable build in a container using: 
```shell script
./mvnw package -Pnative -Dquarkus.native.container-build=true
```

You can then execute your native executable with: `./target/demo-quarkus-1.0.0-SNAPSHOT-runner`

If you want to learn more about building native executables, please consult https://quarkus.io/guides/maven-tooling.html.

## Maven settings

It is recommended to use vanilla maven settings(no custom mirror, proxy) for better performance. If you have modified your default settings `~/.m2/settings.xml` please revert it, or run the maven commands with the clean settings included in this project using  `-s ./settings.xml`

## Deploy to kubernetes

Create your k3d cluster and registry
```shell
k3d registry create registry --port 5000
k3d cluster create -c k8s/dev.yaml
```

Package your app as docker container and push to local k3d registry:
> **_NOTE:_**  Be sure to package your app as native before
```shell
docker build -f src/main/docker/Dockerfile.jvm . -t k3d-registry:5000/demo-quarkus:latest
docker push k3d-registry:5000/demo-quarkus:latest
```

If `push` fails because of unresolved host, you can add it manually (`c:\windows\system32\drivers\etc\hosts` on Windows or `/etc/hosts` on Linux)
```shell
127.0.0.1 k3d-registry
```

Then apply the k8s resources to your cluster(make sure your kubectl has the correct context first)

```shell
kubectl apply -f k8s/postgres-deployment.yaml
kubectl apply -f k8s/postgres-service.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

Give it a few moments and then go to http://demo-quarkus.localhost

## Helm

Check our helm chart and update dependencies.
```
helm lint helm/
helm dependency update helm/
```

Deploy helm chart in the k8s cluster
> **_NOTE:_**  Be sure to remove your old resources
```shell
kubectl delete Service demo-quarkus
kubectl delete Deployment demo-quarkus
kubectl delete Ingress demo-quarkus
```
```
helm install demo-quarkus ./helm
helm list
```

We can also package helm as artefact for the helm repository:
```
helm package helm/
Successfully packaged chart and saved it to: .../reference-project/demo-quarkus-1.0.0.tgz
```

## Istio

Follow the instruction [here](https://istio.io/latest/docs/setup/getting-started/) 
to download and setup istio to the running cluster, as well as clean up resources. Below is just an example

Download and install istio 
```
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/bin:$PATH
istioctl install --set profile=default -y
kubectl label namespace default istio-injection=enabled
kubectl get nodes,svc,deployments,pods,ingress --all-namespaces
```

Deploy gateway and virtual service. Delete svc `traefik` so that istio can configure external IP properly
```
kubectl delete svc traefik -n istio-system
kubectl apply -f k8s/istio/gateway.yaml
```

Test the app.

As an example the virtual service is configured to only handle urls which match `/animals` or `/q/health/ready`. As that a request to `/q/health/live` will return 404

`curl` with `-HHost:demo-quarkus.localhost` if host is specified

Determine host and port if not existed.
```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
echo http://$INGRESS_HOST:$INGRESS_PORT
curl -s http://$INGRESS_HOST:$INGRESS_PORT/animals | python3 -m json.tool
curl --header "Content-Type: application/json" --request POST \
--data '{"name":"dog","basicInfo":"home pet", "numberOfLegs":4}' \
http://$INGRESS_HOST:$INGRESS_PORT/animals
curl -s -I http://$INGRESS_HOST:$INGRESS_PORT/q/health/live
```

Deploy monitoring addon
```
kubectl apply -f k8s/istio/addons/
istioctl dashboard prometheus
istioctl dashboard grafana
istioctl dashboard kiali
for i in $(seq 1 1000); do curl -s -o /dev/null "http://$INGRESS_HOST:$INGRESS_PORT/animals"; done
```

Clean up
```
istioctl manifest generate --set profile=default | kubectl delete --ignore-not-found=true -f -
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
```

## Keycloak
Deploy keycloak, service and ingress. 
```
kubectl apply -f  k8s/keycloak/keycloak.yaml
kubectl apply -f  k8s/keycloak/keycloak-ingress.yaml
curl http://GATEWAY-IP:GATEWAY-PORT/auth/ -HHost:keycloak-demo-quarkus.localhost
```
Go to http://keycloak-demo-quarkus.localhost and log in with `user=password=admin`

Configure keycloak by importing the k8s/keycloak/realm-export.json

    * realm demo-quarkus
    * client demo-quarkus-cli with the url of demo-quarkus app 

As users can not be imported per ui. Please create 3 users for testing

    * 2 roles: user and admin
    * username = u1, password = u1, role_mapping = admin
    * username = u2, password = u2, role_mapping = user
    * username = u3, password = u3

Deploy authorization policy and test

    For testing purpose:

        * [GET] requests to /animals, /animal/{id}, /animal/{id}/facts are only valid if they come from users with role `user`
        * [POST, DELETE] requests to /animals, /animal/{id} are only valid if they come from users with role `admin`
        * requests to /q/health/ready do not need jwt
```
kubectl apply -f k8s/istio/authorization-policy.yaml
curl http://GATEWAY-IP:GATEWAY-PORT/animals -s --header "Authorization: Bearer $TOKEN" | python3 -m json.tool
```








