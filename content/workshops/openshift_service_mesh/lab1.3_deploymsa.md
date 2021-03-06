---
title: Deploying an App
workshops: openshift_service_mesh
workshop_weight: 13
layout: lab
---

# Deploying an App into the Service Mesh

It's time to deploy your microservices application.  The application you are working on is a paste board application in which users can post comments in shared boards.  Here is a diagram of the architecture:


<img src="../images/architecture-highlevel.png" width="800"><br/>

*App Architecture*

The microservices include single sign-on (SSO), user interface (UI), the boards application, and the context scraper.  In this scenario, you are going to deploy these services and then add a new user profile service.


## Deploy Microservices

<blockquote>
<i class="fa fa-terminal"></i>
Navigate to the workshop directory:
</blockquote>

```
cd $HOME/openshift-microservices/deployment/workshop
```

<blockquote>
<i class="fa fa-terminal"></i>
Create a new project for your application.  Use your unique username for the project name:
</blockquote>

```
PROJECT_NAME=<enter username>
oc new-project $PROJECT_NAME --display-name="OpenShift Microservices Demo"
```

You need to add this project to the service mesh.  This is called a [Member Roll][1] resource.  If you do not add the project to the mesh, the microservices will not be added to the service mesh.

<blockquote>
<i class="fa fa-terminal"></i>
Create the member roll resource for your project:
</blockquote>

```
oc apply -f - <<EOF
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members:
    - $PROJECT_NAME
EOF
```

You are going to build the application images from source code and then deploy the resources in the cluster.

The source files are labeled '{microservice}-fromsource.yaml'.  In each file, an annotation 'sidecar.istio.io/inject' was added to tell Istio to inject a sidecar proxy.

<blockquote>
<i class="fa fa-terminal"></i>
Verify the annotation in the 'app-ui' file:
</blockquote>

```
cat openshift-configuration/app-ui-fromsource.yaml | grep -B 1 sidecar.istio.io/inject
```

Output:
```
	annotations:
	  sidecar.istio.io/inject: "true"
```

Now let's deploy the microservices.

<blockquote>
<i class="fa fa-terminal"></i>
Deploy the boards service:
</blockquote>

```
oc new-app -f ./openshift-configuration/boards-fromsource.yaml \
  -p APPLICATION_NAME=boards \
  -p NODEJS_VERSION_TAG=8-RHOAR \
  -p GIT_URI=https://github.com/dudash/openshift-microservices.git \
  -p GIT_BRANCH=workshop-stable \
  -p DATABASE_SERVICE_NAME=boards-mongodb \
  -p MONGODB_DATABASE=boardsDevelopment
```

<blockquote>
<i class="fa fa-terminal"></i>
Deploy the context scraper service:
</blockquote>

```
oc new-app -f ./openshift-configuration/context-scraper-fromsource.yaml \
  -p APPLICATION_NAME=context-scraper \
  -p NODEJS_VERSION_TAG=8-RHOAR \
  -p GIT_BRANCH=workshop-stable \
  -p GIT_URI=https://github.com/dudash/openshift-microservices.git
```

<blockquote>
<i class="fa fa-terminal"></i>
Deploy the user interface:
</blockquote>

```
oc new-app -f ./openshift-configuration/app-ui-fromsource.yaml \
  -p APPLICATION_NAME=app-ui \
  -p NODEJS_VERSION_TAG=8-RHOAR \
  -p GIT_BRANCH=workshop-stable \
  -p GIT_URI=https://github.com/dudash/openshift-microservices.git
```

<blockquote>
<i class="fa fa-terminal"></i>
Deploy single sign-on:
</blockquote>

```
oc new-app -f ./openshift-configuration/sso73-x509-https.yaml \
  -p APPLICATION_NAME=sso \
  -p MEMORY_LIMIT=512Mi
```

<blockquote>
<i class="fa fa-terminal"></i>
Watch the microservices demo installation:
</blockquote>

```
oc get pods -n $PROJECT_NAME --watch
```

Wait a couple minutes.  You should see the 'app-ui', 'boards', 'context-scraper', and 'sso' pods running.  For example:

```
NAME                       READY   STATUS      RESTARTS   AGE
app-ui-1-build             0/1     Completed   0          64m
app-ui-1-xxxxx             2/2     Running     0          62m
app-ui-1-deploy            0/1     Completed   0          62m
boards-1-xxxxx             2/2     Running     0          62m
boards-1-build             0/1     Completed   0          64m
boards-1-deploy            0/1     Completed   0          62m
boards-mongodb-1-xxxxx     2/2     Running     0          64m
boards-mongodb-1-deploy    0/1     Completed   0          64m
context-scraper-1-build    0/1     Completed   0          64m
context-scraper-1-xxxxx    2/2     Running     0          62m
context-scraper-1-deploy   0/1     Completed   0          62m
sso73-x509-1-xxxxx         2/2     Running     0          62m
sso73-x509-1-deploy        1/1     Completed   0          62m
```

Each microservices pod runs two containers: the application itself and the Istio proxy.

<blockquote>
<i class="fa fa-terminal"></i>
Print the containers in the 'app-ui' pod:
</blockquote>

```
oc get pods -l app=app-ui -o jsonpath='{.items[*].spec.containers[*].name}{"\n"}'
```

Output:
```
app-ui istio-proxy
```

## Access Application

The application is deployed!  But you need a way to access the application via the user interface.

Istio provides a [Gateway][2] resource, which is a load balancer at the edge of the service mesh that accepts incoming connections.  You need to deploy a Gateway resource and configure it to route to the application user interface.

<blockquote>
<i class="fa fa-terminal"></i>
Create the gateway load balancer:
</blockquote>

```
oc process -f ./istio-configuration/ingress-loadbalancer.yaml \
  -p INGRESS_GATEWAY_NAME=demogateway | oc create -f -
```

<blockquote>
<i class="fa fa-terminal"></i>
Create the gateway configuration and routing rules:
</blockquote>

```
oc process -f ./istio-configuration/ingress-gateway.yaml \
  -p INGRESS_GATEWAY_NAME=demogateway | oc create -f -
```

To access the application, you need the endpoint of the load balancer you created.

<blockquote>
<i class="fa fa-terminal"></i>
Retrieve the URL of the load balancer:
</blockquote>

```
GATEWAY_URL=$(oc get route istio-demogateway -o jsonpath='{.spec.host}')
echo $GATEWAY_URL
```

<blockquote>
<i class="fa fa-desktop"></i>
Navigate to this URL in the browser.  For example:
</blockquote>

```
https://istio-demogateway-microservices-demo.apps.cluster-naa-xxxx.naa-xxxx.example.opentlc.com:6443
```

You should see the application user interface.  Try creating a new board and posting to the shared board.

For example:

<img src="../images/app-pasteboard.png" width="1024"><br/>
 *Create a new board*

Congratulations, you installed the microservices application!

[1]: https://docs.openshift.com/container-platform/4.1/service_mesh/service_mesh_install/installing-ossm.html#ossm-member-roll_installing-ossm
[2]: https://istio.io/docs/reference/config/networking/gateway/

{{< importPartial "footer/footer.html" >}}