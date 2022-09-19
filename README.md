Overview
![image](https://user-images.githubusercontent.com/67620457/177237647-ccc6ad60-afcd-4235-9fa6-cae8f08d8f8c.png)


### CI
Used Github actions to do CI for backend app at `https://github.com/tan-demo/backend/blob/master/.github/workflows/main.yml`

![image](https://user-images.githubusercontent.com/67620457/177261986-84afdfa0-219b-4706-a184-5c455fc37bb4.png)

- Build docker image and push to ECR. Secrets stored in Github to get login to ECR
- Every push will be tagged testing.SHA (e.g testing.46ebfedbaf2a0ecb7eaa27358afdba267520f8ba
- Every github tag will be tagged with same name (e.g prod.xx.yy.zz)
- Every pull request will be tagged SHA (e.g 46ebfedbaf2a0ecb7eaa27358afdba267520f8ba) for unit test or other test purpose and not push image to ECR

### CD
Used Argocd to deploy applications to EKS cluster. The target will cover the configuration, image changes and no need to build CI again
- Used Argocd Image Updater to check latest image, regexp on ECR then commit to helm values in github
- Github webhook will push event to Argocd and Argocd deploy to EKS
- Used Dex to SSO login from Argocd to Github

### Infra

![image](https://user-images.githubusercontent.com/67620457/177238028-d047c015-0727-44d9-af16-be6a14ade781.png)
![image](https://user-images.githubusercontent.com/67620457/177237987-990131a7-0a0a-4275-96bf-7c0a6accda99.png)

Used 4 modules to build infra
- VPC: create 2 subnet, private and public and NAT gateway
- EKS: Create a cluster, create a iam-assumable-role-with-oidc mapping for the backend to get secret from Secret Manager
- Ingress: Create a Ingress Nginx Controller, load balancer will create automatically
- Argocd: Create argocd application on EKS, set ingress to public https://cd.tanphandns.top, set SSO login, set webhook event, set permission for SSO account login

### Instruction

1. Go to terraform/env/testing 
- Terraform init
- Terraform plan
- Terraform apply
2. Issue Certificate from ACM and config with your domain. If you use aws Route53 or Cloudflare please install external-dns to sync automatically the records
3. Config loadbalancer with your domain
4. In my case, go to https://cd.tanphandns.top then connect to Repository, token is already set in argocd terraform module
![image](https://user-images.githubusercontent.com/67620457/177251829-cb1eb170-2852-4620-8071-66a7fda9d6df.png)
5. Create parent application using UI or code
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: applications-testing
spec:
  destination:
    name: ''
    namespace: ''
    server: 'https://kubernetes.default.svc'
  source:
    path: env/testing
    repoURL: 'https://github.com/tan-demo/applications'
    targetRevision: HEAD
  project: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```
Now you can see all applications will create automatically. It is set by helm chart in repo. Applications are needed for deployment the app
![image](https://user-images.githubusercontent.com/67620457/177252220-26737871-a7cc-4013-80e0-1e06df84bccd.png)
- Argocd Updater: Check new image on ECR and commit to repo helm values by tag (this application was customized to generate ECR token after 10h, the invterval default 2m to 10s)
- Cluster-Autoscaler: This is application to set auto scaling for the cluster
- External-secret: Get secret from ECR through iam assumable role with oidc
- Metric Server: HPA scaling for pod
- Monitoring: Included Prometheus, Grafana and Alertmanager to send a message via email/slack when over thresolds (https://prometheus.tanphandns.top/ https://grafana.tanphandns.top/ https://alert.tanphandns.top/). In this section, the alertmanager will send a simple message to email
![image](https://user-images.githubusercontent.com/67620457/177253642-4bb119fd-21c3-4d57-ad49-5d9a39858ff4.png)
- Nat, Postgresql, Redis: This is used for backend-app.
- Backend-app: This is  API gateway application (source https://techhay.vn/kubernetes-practice-trien-khai-nodejs-microservice-tren-kubernetes-phan-1/). Customzie to helm chart to connect DB and Redis. See the logs that connected to Redis and Postgresql
```
yarn run v1.19.1
59
warning package.json: No license field
58
$ moleculer-runner --config dist/moleculer.config.js
57
[2022-07-05T03:53:23.080Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Moleculer v0.14.13 is starting...
56
[2022-07-05T03:53:23.278Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Namespace: <not defined>
55
[2022-07-05T03:53:23.279Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Node ID: backend-testing-backend-app-545556658d-46hfw-28
54
[2022-07-05T03:53:23.280Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/REGISTRY: Strategy: RoundRobinStrategy
53
[2022-07-05T03:53:23.280Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/REGISTRY: Discoverer: LocalDiscoverer
52
[2022-07-05T03:53:24.078Z] WARN  backend-testing-backend-app-545556658d-46hfw-28/CACHER: The 'redlock' package is missing. If you want to enable cache lock, please install it with 'npm install redlock --save' command.
51
[2022-07-05T03:53:24.079Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Cacher: RedisCacher
50
[2022-07-05T03:53:24.079Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Serializer: JSONSerializer
49
[2022-07-05T03:53:24.182Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Validator: FastestValidator
48
[2022-07-05T03:53:24.280Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Registered 14 internal middleware(s).
47
[2022-07-05T03:53:24.281Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: Transporter: NatsTransporter
46
[2022-07-05T03:53:28.181Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API: API Gateway server created.
45
[2022-07-05T03:53:28.181Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API: Register route to '/api'
44
[2022-07-05T03:53:29.079Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API: ♻ Generate aliases for '/api' route...
43
[2022-07-05T03:53:29.081Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:
42
[2022-07-05T03:53:29.081Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/TRANSIT: Connecting to the transporter...
41
[2022-07-05T03:53:29.980Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/CACHER: Redis cacher connected.
40
[2022-07-05T03:53:30.080Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/TRANSPORTER: NATS client is connected.
39
[2022-07-05T03:53:30.381Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/REGISTRY: Node 'backend-testing-backend-app-categories-745c88bcc-7m8rt-28' connected.
38
[2022-07-05T03:53:30.381Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/REGISTRY: Node 'backend-testing-backend-app-news-d95fc6cdb-2cshv-28' connected.
37
[2022-07-05T03:53:30.979Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/REGISTRY: '$node' service is registered.
36
[2022-07-05T03:53:30.979Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/$NODE: Service '$node' started.
35
[2022-07-05T03:53:30.980Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API: API Gateway listening on http://0.0.0.0:3000
34
[2022-07-05T03:53:30.981Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/REGISTRY: 'api' service is registered.
33
[2022-07-05T03:53:31.178Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API: Service 'api' started.
32
[2022-07-05T03:53:31.180Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/BROKER: ✔ ServiceBroker with 2 service(s) is started successfully in 2s.
31
[2022-07-05T03:53:31.679Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API: ♻ Generate aliases for '/api' route...
30
[2022-07-05T03:53:31.681Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:      GET /api/categories => categories.list
29
[2022-07-05T03:53:31.682Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:     POST /api/categories => categories.create
28
[2022-07-05T03:53:31.778Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:      PUT /api/categories => categories.update
27
[2022-07-05T03:53:31.778Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:      GET /api/news => news.list
26
[2022-07-05T03:53:31.778Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:     POST /api/news => news.create
25
[2022-07-05T03:53:31.778Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:      PUT /api/news/publish => news.publish
24
[2022-07-05T03:53:31.779Z] INFO  backend-testing-backend-app-545556658d-46hfw-28/API:      GET /api/api/list-aliases => api.listAliases
23
22
Sequelize CLI [Node: 12.13.0, CLI: 6.2.0, ORM: 6.6.2]
21
20
Loaded configuration file "migrate/config.js".
19
Using environment "testing".
18
== 20210610065431-create-categories-table: migrating =======
17
== 20210610065431-create-categories-table: migrated (0.200s)
16
15
== 20210804103901-create-news-table: migrating =======
14
== 20210804103901-create-news-table: migrated (0.100s)
13
12
null
11
Sequelize CLI [Node: 12.13.0, CLI: 6.2.0, ORM: 6.6.2]
10
9
Loaded configuration file "migrate/config.js".
8
Using environment "testing".
7
== 20210610065431-create-categories-table: migrating =======
6
== 20210610065431-create-categories-table: migrated (0.200s)
5
4
== 20210804103901-create-news-table: migrating =======
3
== 20210804103901-create-news-table: migrated (0.100s)
2
1
```
### Automations Process:
- Change and commit in github backend repo https://github.com/tan-demo > Github Actions build docker image and push to ECR
- Argocd Image Updater will check new image tag and commit to helm value in https://github.com/tan-demo/applications/blob/main/apps/backend/.argocd-source-backend-testing.yaml
- Argocd will depoly new image to EKS cluster then inform the message to slack chanel (if Argocd Notifications is installed & configured)
- HPA: using Mectric Server for scaling pod. Used busybox to test high load
- Node scaling: using cluster-autoscaler for node https://github.com/kubernetes/autoscaler/tree/master/charts/cluster-autoscaler

### Health check
Implemented for backend deployment and others apps
Example:
```
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 6
            successThreshold: 1
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 6
            successThreshold: 1
```            
