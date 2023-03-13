# kong-poc
For ALB Controller, use aws official document

KONG INGRESS CONTROLLER

KONGA:-
Konga is a fully featured open source, multi-user GUI, that makes the hard task of managing multiple Kong installations a breeze.
Kong Manager (konga) is the graphical user interface (GUI) for Kong Gateway. It uses the Kong Admin API under the hood to administer and control Kong Gateway.

Introduction

Kong is an open-source Lua application using Nginx to have an API gateway in front of your APIs. Thanks to Kong plugins, you can customize the API gateway with many rules like rate-limiting, oauth2 … You can see all the possibilities in the Kong hub.
Kong Ingress Controller allows users to manage the routing rules that control external user access to the service in a Kubernetes cluster from the same platform.
We can deploy kong ingress controller using Helm chart or kubectl manifest files.
Refere following link to deploy kong ingress controller. ( https://github.com/Kong/kubernetes-ingress-controller )
But before that we need to configure application load balancer (ALB) ingress controller.
AWS ALB Ingress Controller for Kubernetes is a controller that triggers the creation of an Application Load Balancer and the necessary supporting AWS resources whenever an Ingress resource is created on the cluster with the kubernetes.io/ingress.class: alb annotation. The Ingress resource uses the ALB to route HTTP or HTTPS traffic to different endpoints within the cluster. 

Problem Statement

Route Application Load balancer traffic through kong ingress controller to the application pod.
			alb -> kong ingress-> service -> pod
      
The problem in kong ingress controller is that, It create network load balancer or classical load balancer at the time of  creating ingress controller. That why it little bit complicated with  that.

Solution Approach


a) Deploy the alb-ingress-controller 
   to install the alb-ingress-controller can be found here (I used kubectl manifest): 
   		( https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html )


b) Deploy the kong-proxy
   Deploy kong without creating a load balancer (use NodePort type). I used kubectl manifest.
   		( https://github.com/Kong/kubernetes-ingress-controller ) 


c) Create your ingress
   Then create ingress pointing to the kong proxy service.


Prerequisites
A server running with Ubuntu 20.04 or later
A running AWS EKS Cluster
Server configured with Docker, Kubectl, AWS CLI v2, eksctl etc.


Traffic Flow:- 
In the Ingress resource for ALB ingress controller, we will add backend as kube-proxy service.
So any request came at ALB DNS it will route traffic to kong ingress controller.
And in kong ingress resource we will add backend as a applications services. Then services will route traffics to pods/container service.


AWS ALB Controller (Installation Setup):- 

Follow below Documentation to install AWS ALB Controller:- 
First Create an IAM OIDC provider for your cluster then create aws alb-controller.
		(https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html )

Official AWS document for installing aws alb-controller: 
		( https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html )

Now create Kong Ingress controller.
 
A Kubernetes Kong deployment contains an ingress controller for managing network traffic. You have different possibilities to deploy it with YAML manifest directly or with the helm chart.

Refere following link to deploy Kong ingress controller.
		( https://github.com/Kong/kubernetes-ingress-controller )

The kong data plane that handles the actual API proxy itself applying all cross-cutting concerns configured for the given API. This is built over Nginx as the base
The kong control plane that receives the configuration on how to proxy an API and persist the same. Kong comes with two different persistence model.
          1) Kong DB Less Mode 
          2) Kong DB Mode
	  
	  
The hearth of this configuration is the Kong Ingress Controller. It aims to manage all API gateway routes.
To store its configuration, we are going to use a PostgreSQL database.
For information, you can deploy a Kong API gateway without a database. You have to manage the configuration in another way, like a configmap.
In this blog, I am going to use Kong API gateway without a Database.


Kong Ingress Controller (Installation Setup)

To install Kong ingress controller, We need to copy the “all-in-one-dbless.yaml”  file from  below link and make some changes according to our requirements.
(https://github.com/Kong/kubernetes-ingress-controller/blob/main/deploy/single/all-in-one-dbless.yaml )


As of configuration of kong, it will create load balancer of type Network / Classic Load Balancer. But we are going to use Application load balancer which we have created from ALB Ingress Controller. So, for that we need to change the service type of kong-poxy Service to NodePort/ ClusterIP . From this Kong ingress controller will not create load balancer of NLB / CLB.

Apply yaml file with all changes.
Verify that the controller is installed.
kubectl get deployment -n <namespace> | grep kong 

	
The output is shown as follows:- 
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
Ingress-kong                             1/1                 1                      1                  84s
     
	
It will create 1 deployment with named ingress-kong in which 2 pods are there and service named kong-proxy. 
Check the status of pods and check logs.
If pods are in running state and in logs its shows successfully reconcile, It means kong ingress controller are successfully deployed on our eks cluster.
After this we need to create ingress resources for ALB as well as Kong Ingress controller.
	
	
Ingress Resource for ALB :-
In this manifest file, we will add backend as the kong ingress service called kube-poxy and route traffic to this service. 
Means whenever request came on alb dns, It will communicate with kong ingress controller.
You can add annotations to kubernetes Ingress and Service objects to customize their behavior.
        
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: alb-ingress-kong
          namespace: kong
          annotations:
            alb.ingress.kubernetes.io/group.name: staging-lb
            # alb.ingress.kubernetes.io/security-groups: sg-03c3fd841ed4c10b4
            # alb.ingress.kubernetes.io/subnets: subnet-05fe57196ad8743a6,subnet-0c0e07b909d2b3d79
            alb.ingress.kubernetes.io/scheme: internet-facing
            alb.ingress.kubernetes.io/healthcheck-port: "80"
            alb.ingress.kubernetes.io/target-type: instance
            # alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":9090}]'
            alb.ingress.kubernetes.io/healthcheck-protocol: HTTP 
            alb.ingress.kubernetes.io/healthcheck-port: traffic-port 
            alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
            alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
            alb.ingress.kubernetes.io/success-codes: 200,404,403,300
            alb.ingress.kubernetes.io/healthy-threshold-count: '2'
            alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
        spec:
          ingressClassName: alb
          rules:   
          - http:
              paths: 
              - backend:
                  service:
                    name: kong-proxy
                    port:
                      number: 80
                path: /
                pathType: Prefix

	
Ingress Resource for Kong Load Balancer:- 
After creating Ingress resource for ALB, we need to create kong ingress resource manifest file in which we route traffic to the application/pod service.
To add multiple path based routing we need to add following annotation. konghq.com/strip-path: 'true' 
        
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: kong-ingress-kong
          namespace: kong
          annotations:
              konghq.com/strip-path: 'true'
              # konghq.com/plugins: rate-limiting-example
        spec:
          ingressClassName: kong
          rules:
          - http:
              paths: 
              - backend:
                  service:
                    name: konga
                    port:
                      number: 1337
                path: /
                pathType: Prefix

Debugging
a) If we want to use existing Application Load Balancer then just add following annotation in ALB ingress resource.
   alb.ingress.kubernetes.io/group.name: <Value>.
b) This annotation should be in both ingress resources from which Application Load Balancer created and also in the ingress resource, which we are going to mentioned backend as a kong-proxy service.
By this, we able to use existing Application Load Balancer

Conclusion
With all these setup, we are able to route Application Load balancer traffic through kong ingress controller to the application pod/service.




Kong Ingress Controller with DB along with Konga

Kong uses an external datastore to store its configuration such as registered APIs, Consumers and Plugins. Plugins themselves can store every bit of information they need to be persisted, for example rate-limiting data or Consumer credentials.
In our case, we will be using PostgresSQL.

PostgreSQL is an established SQL database for use with Kong.
It is a good candidate for single instance or centralized setups due to its relative simplicity and strong performance. Many cloud providers can host and scale PostgreSQL instances, most notably Amazon RDS.
	
In this poc, we are going to use Amazon RDS postgresSQL. 
	
We are going to take all-in-one-postgres.yaml file from following repository.( https://github.com/Kong/kubernetes-ingress-controller/blob/main/deploy/single/all-in-one-postgres.yaml )
	
From that manifest file, remove the statefulset file for postgres application as well as service file.
Then add required environment variable for kong to integrate with amazon RDS. like KONG_PG_USER,	KONG_PG_DATABASE,	KONG_PG_HOST,	KONG_PG_PASSWORD etc.
	
We also make changes in environment variable of CONTROLLER_KONG_ADMIN_URL, KONG_ADMIN_LISTEN from localhost to 0.0.0.0 to configure konga.
After these changes execute the manifest file. 

	
Database Bootstrap
Before starting Kong, we need to bootstrap the database, meaning kong needs to create some tables in the database before it starts. And this work will do our jobs from manifests.

After this we will deploy konga.
We need to create deployment file as well as service manifest for konga.
        
Deployment.yaml:- 
	
	
        apiVersion: apps/v1
        kind: Deployment
        metadata:
         name: konga
         namespace: kong
        spec:
         replicas: 1
         selector:
           matchLabels:
             app: konga
         template:
           metadata:
             labels:
               name: konga
               app: konga
           spec:
             containers:
               - name: konga
                 image: pantsel/konga:0.14.6
                 env:
                   # - name: TOKEN_SECRET
                   #   value: "postgres"
                   - name: DB_HOST
                     value: <>
                   - name: DB_PORT
                     value: <>
                   - name: DB_USER
                     value: <>
                   - name: DB_PASSWORD
                     value: <>
                   - name: DB_DATABASE
                     value: <>
                   - name: DB_ADAPTER
                     value: "postgres"
                   - name: NODE_ENV
                     value: "development"
                 ports:
                   - name: main
                     containerPort: 1337
                     protocol: TCP
        ---
        apiVersion: v1
        kind: Service
        metadata:
         name: konga
         namespace: kong
         annotations:
        spec:
         type: NodePort
         ports:
           - name: konga
             port: 1337
             targetPort: 1337
             protocol: TCP
         selector:
           app: konga



	
After applying above manifest, we can access konga GUI. 

Rate Limiting plugin-
Rate limit means how many HTTP requests can be made in a given period of seconds, minutes, hours, days, months, or years. If the underlying Service/Route (or deprecated API entity) has no authentication layer, the Client IP address will be used; otherwise, the Consumer will be used if an authentication plugin has been configured.
We can add this plugin on service file or ingress resource file.
Just need to add following annotation in metadata section in service or ingress resource file. 
konghq.com/plugins: < name from metadata >

rate-limiting.yaml:- 
	
	
        apiVersion: configuration.konghq.com/v1
        kind: KongPlugin
        metadata:
         name: rate-limiting-example
         # namespace: kong
        config:
         minute: 12
         # second: 15
         limit_by: ip
         policy: local
        plugin: rate-limiting


Basic-auth plugin-
                  Add Basic Authentication to a Service or a Route with username and password protection. The plugin checks for valid credentials in the Proxy-Authorization and Authorization headers (in that order).
Basic-auth.yaml:- 
	
	
                  apiVersion: configuration.konghq.com/v1
                  kind: KongPlugin
                  metadata:
                   name: basic-auth-example
                  config:
                   hide_credentials: true
                  plugin: basic-auth

After this we need to create consumer and secrets for these authentication.

Consumer will be username from yaml. To create credentials we will used kubectl command.

	
kubectl create secret generic <secret_name> --from-literal=kongCredType=basic-auth --from-literal=username=<username_from_yaml>  
--from-literal=password= <password>

	
We need to create kongconsumer. It is kind in kong. 

Kongconsumer.yaml:- 
	
	
                apiVersion: configuration.konghq.com/v1
                kind: KongConsumer
                metadata:
                 name: user1
                 annotations:
                   kubernetes.io/ingress.class: kong
                username: <u_name>
                credentials:
                 - <secret_name>



