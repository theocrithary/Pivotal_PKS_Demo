## Part 1: How to deploy a Kubernetes cluster using Pivotal Container Services (PKS)

Video URL Part 1: https://youtu.be/8MJqHBCe574

#### Create UAA user with PKS Cluster permissions
```
uaac target https://<pks api url>:8443 --skip-ssl-validation
uaac token client get admin -s <uaa admin secret>
uaac user add demo --emails demo@demo.com -p <password>
uaac member add pks.clusters.admin demo
```

#### Login to PKS
```
pks login -a <pks api url> -u demo -p <password> -k
pks clusters
pks create-cluster demo_k8_cluster --external-hostname demo_k8_cluster --plan small
pks cluster demo_k8_cluster
pks get-credentials demo_k8_cluster
```

#### Kubernetes Cluster
```
kubectl cluster-info
kubectl proxy
```

#### Web UI

http://localhost:8001/ui

use config file located at ~/.kube/config (hint: Mac users CMD + Shift + . displays hidden files in finder)


## Part 2: How to deploy a Docker container using Pivotal Container Services (PKS)

Video URL Part 2: https://youtu.be/

#### Pull an image from Docker Hub
```
docker image ls
docker pull node
docker image ls
```

#### Login to Harbor registry web interface

- Browse to the following URL; https://<harbor IP or FQDN>
- Create a new project (e.g. demo)

#### Push an image to Harbor registry

```
docker login <harbor.domainname.com> (use a Harbor username with permissions to the project created earlier)
docker tag node <harbor.domainname.com>/demo/node
docker push <harbor.domainname.com>/demo/node
```
If you get an x.509 certificate error at this point, then refer to troubleshooting notes at the end of this guide.

#### Create a basic Node.js web app file and save as server.js
```
var http = require('http');

var handleRequest = function(request, response) {
  console.log('Received request for URL: ' + request.url);
  response.writeHead(200);
  response.end('Hello World!');
};
var www = http.createServer(handleRequest);
www.listen(8080);
```

#### Create a Docker image file and save as Dockerfile in the same directory as the server.js file
```
FROM node:9.9.0
EXPOSE 8080
COPY server.js .
CMD node server.js
```

#### Build the Docker image with the Dockerfile and Node.js code saved above
```
docker build -t demo-app .
```

#### Push the demo-app image to Harbor registry
```
docker login <harbor.domainname.com> (use a Harbor username with permissions to the project created earlier)
docker tag demo-app <harbor.domainname.com>/demo/demo-app
docker push <harbor.domainname.com>/demo/demo-app
```

#### Deploy the image to the PKS Kubernetes cluster
```
kubectl run demo-app --image=<harbor.domainname.com>/demo/demo-app --port=8080
```

## Troubleshooting Notes: For self signed certs, you will need to perform two tasks;

### 1) Place the Harbor registry root cert into the cert.d directory on your local machine
- Download the Harbor registry Root certificate from https://<<Harbor IP or FQDN>>/api/systeminfo/getcert
- For Mac users, follow the below steps to add the self signed cert to the docker engine
```
sudo mkdir ~/.docker/certs.d/<harbor.domainname.com>
sudo vi ~/.docker/certs.d/<harbor.domainname.com>/client.cert (copy and paste the contents of downloaded certificate file)
```
Restart the Docker service using the toolbar icon and test with the below command
```
docker push <harbor.domainname.com>/demo/node
```

For all other OS's, refer to below;
https://docs.docker.com/registry/insecure/#use-self-signed-certificates

### 2) Place the Harbor registry root cert into the Kubernetes worker node
```
ssh ubuntu@<Ops Manager Server IP> (use the password defined during OVA deployment)
```
Retrieve the "Uaa Admin User Credentials" from the 'Ops Manager Credentials' tab
```
bosh -e <Ops Manager Director IP> log-in (use the admin user credentials as retrieved above)
bosh -e <Ops Manager Director IP> deployments
bosh -e <Ops Manager Director IP> -d service-instance_<UID> vms
bosh -e <Ops Manager Director IP> -d service-instance_<UID> ssh worker/<UID of Worker VM>
```
Download the Harbor registry Root certificate from https://<Harbor IP or FQDN>/api/systeminfo/getcert
```
sudo mkdir /etc/docker/certs.d/<Harbor IP or FQDN>
sudo vi /etc/docker/certs.d/<Harbor IP or FQDN>/ca.crt (copy and paste the contents of downloaded certificate file)
docker pull <harbor.domainname.com>/demo/demo-app (to test and verify the certificate works)
```
