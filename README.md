# Deployment Example
Maintained by chenggck@hk1.ibm.com

## Deploy Instana standalone server
1. Pre-requisite   
Operation System : RHEL 7.2+ (Other Linux OS support)  
Hardware requirement : 16 cores and 64G Memory and 150G or above 
harddisk space  
Instana package : e.g. OIAPM_1.0.193_EN.tar.gz (https://w3-03.ibm.com/software/xl/download/ticket.wss)  

2. Update the RHEL server to latest level (Optional)  

```bash
subscription-manager register # select pool under your RHEL Linux subscription
#ensure rhel-8-for-x86_64-baseos-rpms enabled under /etc/yum.repos/redhat.repo
#ensure rhel-8-for-x86_64-appstream-rpms enabled under /etc/yum.repos/redhat.repo
yum update -y
```
3. Remove podman and runc come with RHEL 8.x (only docker engine supported)
```bash
yum remove podman
yum remove runc
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf repolist -v
dnf install docker-ce-3:18.09.1-3.el7
```
4. Load Instana images to local server
```bash
cp OIAPM_1.0.193_EN.tar.gz /tmp
tar -zxvf OIAPM_1.0.193_EN.tar.gz
cd ./Instana_APM193
./license.sh #Vivew license information
cd ./docker-installer
./instana images import -f ../images/containers.tar
```
5. Start Instana standalone server installation
```bash
mkdir -p /mnt/data /mnt/metrics /mnt/traces
./instana init -f ./settings.hcl
? [Please choose Instana installation type] single
? [What is your tenant name?] <tenant name> 
? [What is your unit name?] <unit name>
? [Insert your agent key (optional). If none is specified, one is generated which does not allow downloads.]
? [Insert your download key or official agent key (optional).]
? [Insert your sales key] 
? [Insert the FQDN of the host] <FQDN>
? [Where should your data be stored?] /mnt/data
? [Where should your trace data be stored?] /mnt/traces
? [Where should your metric data be stored?] /mnt/metrics
? [Where should your logs be stored?] /var/log/instana
? [Path to your signed certificate file?]
? [Path to your private key file?]
Handle certificates
Setup host environment
Clean docker containers
Create configurations
Run data stores
Migrate data stores
Run components
Check components
Setup environment urls
Run nginx
Initialize tenant unit
Welcome to the World of Automatic Infrastructure and Application Monitoring

https://<FQDN>
E-Mail: admin@instana.local
Password: <password>
```
![Alt text](./pic/instanaserver.png?raw=true)

## Deploy Instana agent on OpenShift
1. Login to Instana using the above email and password
```bash
On the Instana Dashboard page. Click left icon "More" and then choose "Agents"
Click "Installing Instana Agents"
Choose "OpenShift"
Give the cluster name and generate the instana-agent.yaml
Download this file to your workstation
```
2. Deloy the Instana daemonset on your OpenShift platform
```bash
oc login # Login to your OpenShift cluster with your admin account with password
oc create -f ./instana-agent.yaml
oc get pod -n instana-agent # Check if agents are deploy on all nodes
e.g. 
[george@workstation instana]$ oc get pod -n instana-agent
NAME                  READY   STATUS    RESTARTS   AGE
instana-agent-87f76   2/2     Running   0          24h
instana-agent-ft4r8   2/2     Running   1          24h
instana-agent-h85r6   2/2     Running   0          24h
instana-agent-qcv7h   2/2     Running   0          24h
```
3. Check if agents shown on Instana Dashboard
```bash
Go back to the agent page and you would see the number of agents are reporting
```
![Alt text](./pic/agents.png?raw=true)  
## Robot-shop for Observability (transaction tracking)
Reference links  
https://www.instana.com/blog/stans-robot-shop-sample-microservice-application/  
https://github.com/instana/robot-shop

Assumption : OpenShift 4.5+ is up and running
1. Download the Robot-shop repository
```bash
git clone https://github.com/instana/robot-shop
```

2. Deploy Robot-shop using helm chart method  
Refer to this link https://github.com/instana/robot-shop/blob/master/K8s/helm/README.md
```bash
oc login # with account and password
oc new-project robot-shop # Create project to host all applications
cd robot-shop/K8s/helm
helm install --set openshift=true --set redis.storageClassName=ibmc-file-bronze-gid robot-shop --namespace robot-shop .
```
3. Several pods were crashed (Monogodb permission denied on /data/db, rating is not able to change to root, net_admin capabliltiy should be added to security context for mysql deployment) Following workaround are performed as below.
```bash
oc set volume deploy/mongodb --add --mount-path /data/db -t pvc --claim-name mongo-data --claim-size=1G
oc adm policy add-scc-to-user anyuid -z default -n robot-shop
oc adm policy add-scc-to-user privileged -z default -n robot-shop
```
4. Restart the failed pods after above modification
```bash
for i in `oc get pod |egrep -v 'NAME|Running' | awk '{print $1}'`
do
oc delete pod $i --force --grace-period 0
done
oc get pod -n robot-shop # To ensure all the pods are running
e.g. 
[george@workstation instana]$ oc get pod
NAME                        READY   STATUS    RESTARTS   AGE
cart-b4bbc8fff-97qvf        1/1     Running   0          50m
catalogue-8b5f66c98-kgpkw   1/1     Running   0          50m
dispatch-67d955c7d8-d4lb5   1/1     Running   0          50m
mongodb-6f5cc64cb7-bps9r    1/1     Running   0          36m
mysql-764c4c5fc7-gdsr7      1/1     Running   0          14m
payment-67c87cb7d-p5szr     1/1     Running   0          30m
rabbitmq-5bb66bb6c9-csn8c   1/1     Running   0          50m
ratings-94fd9c75b-p4ht2     1/1     Running   0          8m26s
redis-0                     1/1     Running   0          50m
shipping-7d69cb88b-qr8nb    1/1     Running   0          8m26s
user-79c445b44b-l8ng8       1/1     Running   0          50m
web-8bb887476-dnjnb         1/1     Running   0          26m 
```
5. Generate the traffic by scale up the load deployment
```bash
oc scale deploy load --replicas 1
```
6. (Optional for suppress payment Erroneous call)
```bash
Method one : Using OCP build
vi robot-shop/payment/payment.py # Make add the following statement at line 85
    if 'items' in cart:                               <-- add this line
        for item in cart.get('items'):                <-- append four spaces
            if item.get('sku') == 'SHIP':             <-- append four spaces
                has_shipping = True                   <-- append four spaces
oc project robot-shop
oc new-build --binary --strategy=docker --name payment
oc start-build payment --from-dir . -F
oc get is payment -o yaml | grep dockerImageReference # to get the new build image
oc set image deploy/payment payment=<new build image>
#No more incident for robot-shop in Instana Dashboard
Method two : Using GitLab CICD pipeline
git clone https://gitlab-ee.coc-ibm.com/instana-robot-shop/payment.git
cd payment
vi payment.py # Make add the following statement at line 75
    if 'items' in cart:                               <-- add this line
        for item in cart.get('items'):                <-- append four spaces
            if item.get('sku') == 'SHIP':             <-- append four spaces
                has_shipping = True                   <-- append four spaces
git add payment.py
git commit -m "fix the payment service bug"
git push
# GitLab CICD pipeline will be triggered. 
```
7. Define pipeline feedback from Jenkins  
```bash
oc new-project cicd
oc new-app --template=jenkins-persistent
oc get route (Try to access the Jenkins via the URL and ensure you are able to login)
oc adm policy add-role-to-user admin system:serviceaccount:cicd:jenkins -n robot-shop
# Replace token and Instana server URL as below.
cat << 'EOF' > ./pipeline-feedback.yaml
kind: "BuildConfig"
apiVersion: "v1"
metadata:
  name: "pipeline-feedback"
spec:
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfile: |-
        pipeline
        {
          agent {
            label 'master'
          }
          stages
          {
            stage('Git checkout source')
            {
              steps
              {
                sh "echo git clone source"
              }
            }
            stage('Maven build')
            {
              steps
              {
                sh "echo Maven build"
              }
            }
            stage('JUnit Test')
            {
              steps
              {
                sh "echo Unit Test"
              }
            }
            stage('Code Analysis')
            {
              steps
              {
                sh "echo Sonarqube code analysis"
              }
            }
            stage('Archive App') {
              steps {
                sh "echo Archive application"
              }
            }
            stage('Create Image Builder') {
              steps {
                sh "echo Create Image Builder"
              }
            }
            stage('Build Image') {
              steps {
                sh "echo Build Image"
              }
            }
            stage('Deploy DEV') {
              steps {
                sh "oc set env deploy/load --all ERROR='1' -n robot-shop"
              }
            }
            stage('Notifiy Instana') {
              steps {
                script {
                  env.apiToken=<token>
                  env.base=<instana server URL>
                  env.timestamp="${currentBuild.startTimeInMillis}"
                  env.pipelinename="New image deployed"
                  env.service="payment"
                  echo "$env.timestamp"
                }
              }
            }
          }
        }
    type: JenkinsPipeline
EOF
oc create -f ./pipeline-feedback.yaml

```
![Alt text](./pic/robot-shop.png?raw=true)  
Pipeline feedback for CICD defined in Jenkins under OpenShift
![Alt text](./pic/startbuild.png?raw=true)  

## Enable EUM on browser client side
1. Generate Instana EUM reportingUrl and EUM key
```bash
Login to Instana Dashboard
Click "WebSites and Mobile Applications"
Click "Add Website" and give an name "Robot-Shop"
Find the below variables and will be used in later steps
reportingUrl : <reportingUrl>
key: <key>
```
2. Stop web deployment and change the EUM environment variable
```bash
oc login 
oc project robot-shop
oc scale deploy web --replicas 0 
oc edit deploy web #change the following environment in the deployment as below
      - env:
        - name: INSTANA_EUM_KEY
          value: <key>
        - name: INSTANA_EUM_REPORTING_URL
          value: http://<hostname of the onpremise Instana>:86/eum/
```
3. Start web deployment
```bash
oc scale deploy web --replicas 1
```
![Alt text](./pic/eum.png?raw=true)  
## Enable Humio integration with Instana
Assumption : Humio is deployed on the same OpenShift platform
1. Create new repository called 'aiops' on Humio Dashboard and get the ingest token
xxxxxxxx-0dfb-4bdf-84e0-2411eaxxxxxxx

2. Deploy Humio-fluentbit on the OpenShift
Refer to https://docs.humio.com/docs/ingesting-data/log-formats/kubernetes/
```bash
helm repo add humio https://humio.github.io/humio-helm-charts
helm repo update
oc adm policy add-scc-to-user privileged system:serviceaccount:humio:humio-fluentbit-read
helm install humio humio/humio-helm-charts --namespace humio --set humio-fluentbit.token="<token>" --values humio-agent.yaml
#notes : the content of humio-agent.yaml as below
=====
humio-core:
  enabled: false
humio-fluentbit:
  enabled: true
  replicas: 1
  humioRepoName: aiops
  humioHostname: <Humio route exposed by OpenShift> (oc get route -n <Humio namespace)
  es:
    port: 80
    tls: false
=====
(Optional) oc edit daemonset <fluent daemonset>  and make the following change to ensure the fluent process access to hostpath
        securityContext:
          privileged: true
(Optional) oc edit cm humio-fluent-bit-config  (to reduce the log sending to Humio from demo app only)
  fluent-bit-input.conf: |-
    [INPUT]
        Name             tail
        Path             /var/log/containers/*robot-shop*.log
```
3. Integrate Instana with Humio
```bash
Login to Instana Dashboard
Click "Setting" and "Humio"
Input the Humio URL and repository created (e.g. aiops)
```
![Alt text](./pic/humio1.png?raw=true)  
## ServiceNow integration
1. Register as developer in ServiceNow via the below link  
https://developer.servicenow.com/dev.do
2. Request an ServiceNow instance which is free of charge but will be destroyed if no activity for more than 10 days.  
3. Fork https://github.ibm.com/chenggck/servicenow-integration.git into your own repository
4. Login to the ServiceNow instance using admin account  
https://dev<instance id>.service-now.com
5. Deploy the Watson AIOps application to your ServiceNow instance using branch name sn_instances/vendor  
Put "system applications" in the search box of ServiceNow Dashboard  
Click "Studio"  
Click "Import from source control"  
Enter the following information and import the Watson AIOps
URL: https://github.ibm.com/<org|persion>/servicenow-integration.git  
Branch: sn_instances/vendor  
MID Server Name: Optional  
Credential: <developer token of you  
![Alt text](./pic/ServiceNow.png?raw=true)  
## Watson AIOps integration
1. Define application group, application, ASM integration, Slack integration in Watson AIOps Dashboard  
Instance : aiops  
Application Group : robot-shop  
Application : payment  
Slack URL : https://aiops-cpd-aiops.<base url>/aiops/aimanager/instances/1611056417956638/api/slack/events  
Slack channel: < channel id shown in Slack>  
eASM:  
  https://noi-topology-topology.noi.svc:8080  
  https://noi-topology-layout.noi.svc:7084  
  https://noi-topology-merge.noi.svc:7082  
  https://noi-topology-search.noi.svc:7080  
  https://netcool.noi.<baseurl>  
  https://noi-topology-ui-api.noi.svc:3080  
ServiceNow URL: https://devxxxxx.service-now.com  

2. Humio log training
```bash
oc project robot-shop
oc set env deploy/load --all ERROR="0"  #Prevent the load pod to generate payment 
oc set probe deploy/payment --liveness --get-url=http://:8080/health
oc delete pod $(oc get pod |grep payment)
# Wait for one day or less to let Humio to collect health logs for training
export HUMIO_CLUSTER_URL=<HUMIO_CLUSTER_URL> # oc get route -n humio
export API_TOKEN=<API_TOKEN>           # Get it from your Humio Dashboard
export REPOSITORY_NAME=aiops
curl http://$HUMIO_CLUSTER_URL/api/v1/repositories/$REPOSITORY_NAME/query -X POST  -H "Authorization: Bearer $API_TOKEN" -H 'Content-Type: application/json' -H "Accept: application/x-ndjson" -d '{"queryString":"kubernetes.labels.service=payment","start":"24hours","end":"now"}' | head -n 10000 > ./payment_raw.json
jq . payment_raw.json | more           # Ensure health log entries are collected for train
oc project aiops
oc cp ./payment_raw.json $(oc get pod |grep train-console | awk '{print $1}'):/tmp  
oc exec -it $(oc get pod |grep train-console | awk '{print $1}') bash
export application_group_id="zlx2gcpr"         #Look it from Log Model in Watson AIOps GUI
export application_id="slm6vj5m"               #Look it from Log Model in Watson AIOps GUI
aws s3 mb s3://$LOG_INGEST
aws s3 cp /tmp/payment_raw.json s3://$LOG_INGEST/$application_group_id/$application_id/1/payment_raw.json
cd /home/zeno/train
python3 train_pipeline.pyc -p "log" -g "$application_group_id" -a "$application_id" -v "1"
e.g.
<user1>$ python3 train_pipeline.pyc -p "log" -g "zlx2gcpr" -a "slm6vj5m" -v "1"
Launching Jobs for: Log Ingest Training
  prepare training environment
  training_ids:
    1: ['training-CCsW_SsGR']
  jobs ids are saved here: JOBS/zlx2gcpr/slm6vj5m/1/log_ingest.json
  Training... |████████████████████████████████| [0:08:21], complete: 1/1 {'COMPLETED': 1}
  Job-ids ['training-CCsW_SsGR'] are COMPLETED
  Please check the logs of COMPLETED jobs here: s3://log-anomaly

Launching Jobs for: Log Anomaly Training
  prepare training environment
  training_ids:
    1: payment --> ['training-ksw-_SyMg']
  jobs ids are saved here: JOBS/zlx2gcpr/slm6vj5m/1/log_anomaly.json
  Training... |████████████████████████████████| [0:04:20], complete: 1/1 {'COMPLETED': 1}
  Job-ids ['training-ksw-_SyMg'] are COMPLETED
  Please check the logs of COMPLETED jobs here: s3://log-model
  moving trained models
    log-model/training-ksw-_SyMg/zlx2gcpr/slm6vj5m --> log-model/zlx2gcpr/slm6vj5m
Deploying the log pipeline automatically
  updating elastic search database
  informing controller to refresh job
    attempt: 1 response: 200
```
3. Humio to Watson AIOps integration  
Define Humio connection for payment application under Robot-shop application group   

![Alt text](./pic/humio2.png?raw=true)

## Demo use cases
1. Show how Instana provide obserability of microservice
* Application services and end user monitoring on Robot-shop
![Alt text](./pic/instana1.png?raw=true)

* OpenShift platform and Kubernetes monitoring
![Alt text](./pic/instana2.png?raw=true)

* Infrastructure using sensors via agent
![Alt text](./pic/instana3.png?raw=true)

* Transaction tracking with drill down of time spanned
![Alt text](./pic/instana4.png?raw=true)

* Alert definition
![Alt text](./pic/instana5.png?raw=true)

* Technology sensor supported
![Alt text](./pic/instana6.png?raw=true)

2. Pipeline feedback
* Trigger an Jenkins pipeline to rollout new payment microservice (which trigger alert)
![Alt text](./pic/startbuild.png?raw=true)

* Ensure the build complete
![Alt text](./pic/startbuild1.png?raw=true)


* Use Instana to inspect and identify the root cause of the problem.
![Alt text](./pic/instana7.png?raw=true)

* Also, with pipeline feedback defined in CICD pipeline, the "rocket" indicate the chagne on Payment microservice under Robot-Shop application  
![Alt text](./pic/instana8.png?raw=true)

* Show the stack story which is generated by Watson AIOps for this problem using unstructure data machine learning on Payment application logs and the ChatOps integration. Incident is created on Service Now for this story.
![Alt text](./pic/slack1.png?raw=true)
![Alt text](./pic/slack2.png?raw=true)
 
* Service Now ticket is created for this Watson AIOps story.
![Alt text](./pic/servicenow1.png?raw=true)













