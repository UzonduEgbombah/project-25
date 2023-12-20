## Deploying and Packaging applications into Kubernetes with Helm
In the previous project, you started experiencing helm as a tool used to deploy an application into Kubernetes. You probably also tried installing more tools apart from Jenkins.
In this project, you will experience deploying more DevOps tools, get familiar with some of the real world issues faced during such deployments and how to fix them. You will learn how to tweak helm values files to automate the configuration of the applications you deploy. Finally, once you have most of the DevOps tools deployed, you will experience using them and relate with the DevOps cycle and how they fit into the entire  ecosystem.
Our focus will be on the.

- Artifactory

- Ingress Controllers

- Cert-Manager

Then you will attempt to explore these on your own.

- Prometheus
  
- Grafana
  
- Elasticsearch ELK using ECK


For the tools that require paid license, don't worry, you will also learn how to get the license for free and have true experience exactly how they are used in the real world.
Lets start first with Artifactory. What is it exactly?
Artifactory is part of a suit of products from a company called Jfrog. Jfrog started out as an artifact repository where software binaries in different formats are stored. Today, Jfrog has transitioned from an artifact repository to a DevOps Platform that includes CI and CD capabilities. This has been achieved by offering more products in which Jfrog Artifactory is part of. Other offerings include

JFrog Pipelines -  a CI-CD product that works well with its Artifactory repository. Think of this product as an alternative to Jenkins.
JFrog Xray - a security product that can be built-into various steps within a JFrog pipeline. Its job is to scan for security vulnerabilities in the stored artifacts. It is able to scan all dependent code.

In this project, the requirement is to use Jfrog Artifactory as a private registry for the organisation's Docker images and Helm charts. This requirement will satisfy part of the company's corporate security policies to never download artifacts directly from the public into production systems. We will eventually have a CI pipeline that initially pulls public docker images and helm charts from the internet, store in artifactory and scan the artifacts for security vulnerabilities before deploying into the corporate infrastructure. Any found vulnerabilities will immediately trigger an action to quarantine such artifacts.
Lets get into action and see how all of these work.

#### Deploy Jfrog Artifactory into Kubernetes
The best approach to easily get Artifactory into kubernetes is to use helm.

-  Search for an official helm chart for Artifactory on Artifact Hub

-  Add the jfrog remote repository on your laptop/computer

- Create a namespace called tools where all the tools for DevOps will be deployed. (In previous project, you installed Jenkins in the default namespace. You should uninstall Jenkins 
  there and install in the new namespace)

- Update the helm repo index on your laptop/computer

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/183c8b11-eec0-40e3-994c-e30a1bd9b4ad)

- Install artifactory

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/73e6009f-6295-4c03-bc06-818561f0e817)

#### NOTE:
We have used upgrade --install flag here instead of helm install artifactory jfrog/artifactory This is a better practice, especially when developing CI pipelines for helm deployments. It ensures that helm does an upgrade if there is an existing installation. But if there isn't, it does the initial install. With this strategy, the command will never fail. It will be smart enough to determine if an upgrade or fresh installation is required.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/7a34f316-f8c1-4426-8d3e-b033b61b1cde)



#### Getting the Artifactory URL
Lets break down the first Next Step.

The artifactory helm chart comes bundled with the Artifactory software, a PostgreSQL database and an Nginx proxy which it uses to configure routes to the different capabilities of Artifactory. Getting the pods after some time, you should see something like the below.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/a2aace3e-3d3e-4a72-8ad3-89c1384097b2)

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/07b492e3-7a0b-4b45-ab9c-d418d2df5ff4)

- Each of the deployed application have their respective services. This is how you will be able to reach either of them.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/a1460f33-008f-47a8-859c-699782db0770)

- Notice that, the Nginx Proxy has been configured to use the service type of LoadBalancer. Therefore, to reach Artifactory, we will need to go through the Nginx proxy’s service. Which 
  happens to be a load balancer created in the cloud provider. Run the kubectl command to retrieve the Load Balancer URL.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/eb822588-cddb-4def-af7b-603a6b770a3f)

- Copy the URL and paste in the browser

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/190cfd33-7eb5-4191-abab-ddd226f11a48)

- The default username is admin

- The default password is password

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/b69fd47d-e84c-49de-abf2-1e33f2cf172d)

#### How the Nginx URL for Artifactory is configured in Kubernetes
Without clicking further on the Get Started page, lets dig a bit more into Kubernetes and Helm. How did Helm configure the URL in kubernetes?

Helm uses the values.yaml file to set every single configuration that the chart has the capability to configure. THe best place to get started with an off the shelve chart from artifacthub.io is to get familiar with the DEFAULT VALUES

- click on the DEFAULT VALUES section on Artifact hub

- Here you can search for key and value pairs

- For example, when you type nginx in the search bar, it shows all the configured options for the nginx proxy.

- selecting nginx.enabled from the list will take you directly to the configuration in the YAML file.

- Search for nginx.service and select nginx.service.type

- You will see the confired type of Kubernetes service for Nginx. As you can see, it is LoadBalancer by default

- To work directly with the values.yaml file, you can download the file locally by clicking on the download icon.

Is the Load Balancer Service type the Ideal configuration option to use in the Real World?
Setting the service type to Load Balancer is the easiest way to get started with exposing applications running in kubernetes externally. But provisioning load balancers for each application can become very expensive over time, and more difficult to manage. Especially when tens or even hundreds of applications are deployed.

The best approach is to use Kubernetes Ingress instead. But to do that, we will have to deploy an Ingress Controller.

A huge benefit of using the ingress controller is that we will be able to use a single load balancer for different applications we deploy. Therefore, Artifactory and any other tools can reuse the same load balancer. Which reduces cloud cost, and overhead of managing multiple load balancers. more on that later.

For now, we will leave artifactory, move on to the next phase of configuration (Ingress, DNS(Route53) and Cert Manager), and then return to Artifactory to complete the setup so that it can serve as a private docker registry and repository for private helm charts.

#### DEPLOYING INGRESS CONTROLLER AND MANAGING INGRESS RESOURCES
Before we discuss what ingress controllers are, it will be important to start off understanding about the Ingress resource.

An ingress is an API object that manages external access to the services in a kubernetes cluster. It is capable of providing load balancing, SSL termination and name-based virtual hosting. In otherwords, Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/c8f17c4c-500e-4550-9710-7ada47a2dc8f)

An ingress resource for Artifactory would like like below


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:
  - host: "tooling.artifactory.egbombah.store"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082

        
An Ingress needs apiVersion, kind, metadata and spec fields
The name of an Ingress object must be a valid DNS subdomain name
Ingress frequently uses annotations to configure some options depending on the Ingress controller.
Different Ingress controllers support different annotations. Therefore it is important to be up to date with the ingress controller’s specific documentation to know what annotations are supported.
It is recommended to always specify the ingress class name with the spec ingressClassName: nginx. This is how the Ingress controller is selected, especially when there are multiple configured ingress controllers in the cluster.
The domain onyeka.ga should be replaced with your own domain.
Ingress controller
If you deploy the yaml configuration specified for the ingress resource without an ingress controller, it will not work. In order for the Ingress resource to work, the cluster must have an ingress controller running.

Unlike other types of controllers which run as part of the kube-controller-manager. Such as the Node Controller, Replica Controller, Deployment Controller, Job Controller, or Cloud Controller. Ingress controllers are not started automatically with the cluster.

Kubernetes as a project supports and maintains AWS, GCE, and NGINX ingress controllers.
There are many other 3rd party Ingress controllers that provide similar functionalities with their own unique features, but the 3 mentioned earlier are currently supported and maintained by Kubernetes. Some of these other 3rd party Ingress controllers include but not limited to the following;

- AKS Application Gateway Ingress Controller (Microsoft Azure)

- Istio

- Traefik

- Ambassador

- HA Proxy Ingress

- Kong

- Gloo
  
An example comparison matrix of some of the controllers can be found here https://kubevious.io/blog/post/comparing-top-ingress-controllers-for-kubernetes#comparison-matrix. Understanding their unique features will help businesses determine which product works well for their respective requirements.

It is possible to deploy any number of ingress controllers in the same cluster. That is the essence of an ingress class. By specifying the spec ingressClassName field on the ingress object, the appropriate ingress controller will be used by the ingress resource.

Lets get into action and see how all of these fits together.

#### Note:
Before deploying the ingress controller, remember to change the nginx.service.type of the artifactory from LoadBalancer to ClusterIP with the below command

helm upgrade --install artifactory jfrog/artifactory --set nginx.service.type=ClusterIP,databaseUpgradeReady=true -n tools

#### Deploy Nginx Ingress Controller
On this project, we will deploy and use the Nginx Ingress Controller. It is always the default choice when starting with Kubernetes projects. It is reliable and easy to use.

Since this controller is maintained by Kubernetes, there is an official guide the installation process. Hence, we wont be using artifacthub.io here. Even though you can still find ready to go charts there, it just makes sense to always use the official guide in this scenario.

Using the Helm approach, according to the official guide;

Install Nginx Ingress Controller in the ingress-nginx namespace
helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx --create-namespace

Notice:
This command is idempotent:

if the ingress controller is not installed, it will install it,

if the ingress controller is already installed, it will upgrade it.

Self Challenge Task – Delete the installation after running above command. Then try to re-install it using a slightly different method you are already familiar with. Ensure NOT to use the flag --repo

Hint – Run the helm repo add command before installation

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/1f2fdf31-b51c-46c6-a7dc-a4227328a594)

- A few pods should start in the ingress-nginx namespace:

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/67fd20de-6da1-43e3-87c2-2791d45e4872)


- After a while, they should all be running. The following command will wait for the ingress controller pod to be up, running, and ready:

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/370d9692-86c4-4c44-b4eb-aeaf456859b8)

- Check to see the created load balancer in AWS.

kubectl get service -n ingress-nginx

Output:

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/c8c62dc7-7ff5-4e2a-863f-4d41f2a45c13)

The ingress-nginx-controller service that was created is of the type LoadBalancer. That will be the load balancer to be used by all applications which require external access, and is using this ingress controller.

If you go ahead to AWS console, copy the address in the EXTERNAL-IP column, and search for the loadbalancer, you will see an output like below.

![Screenshot 2023-12-19 220136](https://github.com/UzonduEgbombah/project-25/assets/137091610/a5ec739c-9c8f-48f5-9837-16b78438574e)

- Check the IngressClass that identifies this ingress controller.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/4952bc7b-4a72-47e2-89cc-760bfab161cd)

#### Deploy Artifactory Ingress
Now, it is time to configure the ingress so that we can route traffic to the Artifactory internal service, through the ingress controller’s load balancer.

Notice the spec section with the configuration that selects the ingress controller using the ingressClassName

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/1f50caad-ea0d-4b74-ac9c-b9d494eaa440)

- kubectl apply -f <filename.yaml> -n tools

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/7a20b528-b528-40b1-9c9b-3f5222409f05)

- kubectl get ing -n tools

Output

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/82cfc0d7-73fa-49d6-9c20-8ddc23ade404)


Now, take note of

- CLASS – The nginx controller class name nginx

- HOSTS – The hostname to be used in the browser tooling.artifactory.onyeka.ga

- ADDRESS – The loadbalancer address that was created by the ingress controller

#### Configure DNS
If anyone were to visit the tool, it would be very inconvenient sharing the long load balancer address. Ideally, you would create a DNS record that is human readable and can direct request to the balancer. This is exactly what has been configured in the ingress object - host: "tooling.tooling.egbombah.store" but without a DNS record, there is no way that host address can reach the load balancer.

The egbombah.store part of the domain is the configured HOSTED ZONE in AWS. So you will need to configure Hosted Zone in AWS console or as part of your infrastructure as code using terraform.

If you purchased the domain directly from AWS, the hosted zone will be automatically configured for you. But if your domain is registered with a different provider such as namechaep, you will have to create the hosted zone and update the name servers.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/37be54dd-6dfd-42cd-8c86-f309e4518d59)


#### Create Route53 record
Within the hosted zone is where all the necessary DNS records will be created. Since we are working on Artifactory, lets create the record to point to the ingress controller’s loadbalancer. There are 2 options. You can either use the CNAME or AWS Alias

- CNAME Method
  
Select the HOSTED ZONE you wish to use, and click on the create record button

Add the subdomain tooling.artifactory, and select the record type CNAME

Successfully created record

Confirm that the DNS record has been properly propergated. Visit https://dnschecker.org and check the record. Ensure to select CNAME. The search should return green ticks for each of the locations on the left.


![](https://github.com/UzonduEgbombah/project-25/assets/137091610/537e5ceb-229e-40a3-bce2-85c9f28c7e80)


- AWS Alias Method

In the create record section, type in the record name, and toggle the alias button to enable an alias. An alias is of the A DNS record type which basically routes directly to the load balancer. In the choose endpoint bar, select Alias to Application and Classic Load Balancer

Select the region and the load balancer required. You will not need to type in the load balancer, as it will already populate.

For detailed read on selecting between CNAME and Alias based records, read the official documentation https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html.

Visiting the application from the browser
So far, we now have an application running in Kubernetes that is also accessible externally. That means if you navigate to https://tooling.artifactory.egbombah.store/ , it should load up the artifactory application.

Using Chrome browser will show something like the below. It shows that the site is indeed reachable, but insecure. This is because Chrome browsers do not load insecure sites by default. It is insecure because it either does not have a trusted TLS/SSL certificate, or it doesn’t have any at all.

Nginx Ingress Controller does configure a default TLS/SSL certificate. But it is not trusted because it is a self signed certificate that browsers are not aware of.

To confirm this,

- Click on the Not Secure part of the browser.

- Select the Certificate is not valid menu

- You will see the details of the certificate. There you can confirm that yes indeed there is encryption configured for the traffic, the browser is just not cool with it.

#### Explore Artifactory Web UI

Now that we can access the application externally, although insecure, its time to login for some exploration. Afterwards we will make it a lot more secure and access our web application on any browser.

Get the default username and password – Run a helm command to output the same message after the initial install
helm test artifactory -n tools
Output:

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/37d04b2e-546e-428b-9f85-660453ad734b)

- Insert the username and password to load the Get Started page

- Reset the admin password

- Activate the Artifactory License. You will need to purchase a license to use Artifactory enterprise features.

- For learning purposes, you can apply for a free trial license. Simply fill the form here https://jfrog.com/start-free/, use the Self-Hosted button and a license key will be delivered 
  to your email in few minutes.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/01bcacfb-94f1-453e-9e81-3dfc78b65341)


- In exactly 1 minute, the license key had arrived. Simply copy the key and apply to the console.

- Set the Base URL. Ensure to use https

- Skip the Proxy setting

- Skip creation of repositories for now. You will create them yourself later on.

- finish the setup


Next, its time to fix the TLS/SSL configuration so that we will have a trusted HTTPS URL

#### DEPLOYING CERT-MANAGER AND MANAGING TLS/SSL FOR INGRESS
Transport Layer Security (TLS), the successor of the now-deprecated Secure Sockets Layer (SSL), is a cryptographic protocol designed to provide communications security over a computer network.

The TLS protocol aims primarily to provide cryptography, including privacy (confidentiality), integrity, and authenticity through the use of certificates, between two or more communicating computer applications.

The certificates required to implement TLS must be issued by a trusted Certificate Authority (CA).

To see the list of trusted root Certification Authorities (CA) and their certificates used by Google Chrome, you need to use the Certificate Manager built inside Google Chrome as shown below:

Open the settings section of google chrome

Search for security

Select Manage Certificates

View the installed certificates in your browser

Certificate Management in Kubernetes
Ensuring that trusted certificates can be requested and issued from certificate authorities dynamically is a tedious process. Managing the certificates per application and keeping track of expiry is also a lot of overhead.

To do this, administrators will have to write complex scripts or programs to handle all the logic.

Cert-Manager comes to the rescue!

cert-manager adds certificates and certificate issuers as resource types in Kubernetes clusters, and simplifies the process of obtaining, renewing and using those certificates.

Similar to how Ingress Controllers are able to enable the creation of Ingress resource in the cluster, so also cert-manager enables the possibility to create certificate resource, and a few other resources that makes certificate management seamless.

It can issue certificates from a variety of supported sources, including Let’s Encrypt, HashiCorp Vault, and Venafi as well as private PKI. The issued certificates get stored as kubernetes secret which holds both the private key and public certificate.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/1c88d583-55b2-43cc-999a-ce3f6bfa9241)

In this project, We will use Let’s Encrypt with cert-manager. The certificates issued by Let’s Encrypt will work with most browsers because the root certificate that validates all it’s certificates is called “ISRG Root X1” which is already trusted by most browsers and servers.

You will find ISRG Root X1 in the list of certificates already installed in your browser.

Read the official documentation here https://letsencrypt.org/docs/certificate-compatibility/

Cert-manager will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry.

#### Cert-Manager high Level Architecture
Cert-manager works by having administrators create a resource in kubernetes called certificate issuer which will be configured to work with supported sources of certificates. This issuer can either be scoped globally in the cluster or only local to the namespace it is deployed to.

Whenever it is time to create a certificate for a specific host or website address, the process follows the pattern seen in the image below.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/cf3a33b6-98e2-4f48-b224-12c721ae371b)


After we have deployed cert-manager, you will see all of this in action.

#### Deploying Cert-manager
Note:
Please refer to https://github.com/onyeka-hub/devops-tools-training.git for proper guildlines on the installation of cert-manager with an appropriate service account.

Before installing the chart, you must first install the cert-manager CustomResourceDefinition resources. This is performed in a separate step to allow you to easily uninstall and reinstall cert-manager without deleting your installed custom resources.

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.crds.yaml

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/9345a35f-ce2f-4fd3-97e8-6e95b5c17570)


-  Add the Jetstack Helm repository
  
helm repo add jetstack https://charts.jetstack.io

-  Install the cert-manager helm chart
  
helm upgrade -i cert-manager --version v1.13.2 jetstack/cert-manager --create-namespace --namespace cert-manager 

You should see an output like this

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/7d0d8f48-ce8a-4490-8960-3a8df5006dc9)


- Verify that the deployment was successful:

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/9bb973f5-c478-4631-846c-bbf191755b1b)


If you want to completely uninstall cert-manager from your cluster, you will also need to delete the previously installed CustomResourceDefinition resources:

kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yam

#### Certificate Issuer
Next, is to create an Issuer. We will use a Cluster Issuer so that it can be scoped globally. Assuming that we will be using onyeka.ga domain. Simply update this cert-manager.yaml file and deploy with kubectl. In the section that follows, we will break down each part of the file.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/bd26bf3d-52eb-47f9-9256-35bd8613c596)

Lets break down the content to undertsand all the sections

Section 1 – The standard kubernetes section that defines the apiVersion, Kind, and metadata. The Kind here is a ClusterIssuer which means it is scoped globally.

apiVersion: cert-manager.io/v1

kind: ClusterIssuer

metadata:

namespace: "cert-manager"

name: "letsencrypt-prod"

Section 2
– In the spec section, an ACME – Automated Certificate Management Environment issuer type is specified here. When you create a new ACME Issuer, cert-manager will generate a private key which is used to identify you with the ACME server.
Certificates issued by public ACME servers are typically trusted by client’s computers by default. This means that, for example, visiting a website that is backed by an ACME certificate issued for that URL, will be trusted by default by most client’s web browsers. ACME certificates are typically free.

Let’s Encrypt uses the ACME protocol to verify that you control a given domain name and to issue you a certificate. You can either use the let’s encrypt Production server address https://acme-v02.api.letsencrypt.org/directory which can be used for all production websites. Or it can be replaced with the staging URL https://acme-staging-v02.api.letsencrypt.org/directory for all Non-Production sites.

The privateKeySecretRef has configuration for the private key name you prefer to use to store the ACME account private key. This can be anything you specify, for example letsencrypt-staging

spec:
 
 acme:
    
    server: "https://acme-v02.api.letsencrypt.org/directory"
    
    email: "uzonduegbombah419@gmail.com"
    
    privateKeySecretRef:
    
    name: "letsencrypt-prod"


section 3

 This section is part of the spec that configures solvers which determines the domain address that the issued certificate will be registered with. dns01 is one of the different challenges that cert-manager uses to verify domain ownership. Read more on DNS01 Challenge here https://letsencrypt.org/docs/challenge-types/#dns-01-challenge. With the DNS01 configuration, you will need to specify the Route53 DNS Hosted Zone ID and region. Since we are using EKS in AWS, the IAM permission of the worker nodes will be used to access Route53. Therefore if appropriate permissions is not set for EKS worker nodes, it is possible that certificate challenge with Route53 will fail, hence certificates will not get issued.
The other possible option is the HTTP01 challenge, but we won’t be using that here.


solvers:

- selector:

  dnsZones:
    
   - "egbombah.store"

dns01:
    
    route53:
    
    region: "us-east-1"
    
    hostedZoneID: "xxxxxxxxx"


- kubectl apply -f clusterissuer.yaml
With the ClusterIssuer properly configured, it is now time to start getting certificates issued.

#### CONFIGURING INGRESS FOR TLS
To ensure that every created ingress also has TLS configured, we will need to update the ingress manifest with TLS specific configurations.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/bbaffaa6-9dac-4fd0-86c2-419df0ee705b)

The most significat updates to the ingress definition is the annotations and tls sections.

Lets quickly talk about Annotations. Annotations are used similar to labels in kubernetes. They are ways to attach metadata to objects.

Differences between Annotations and Labels
Labels are used in conjunction with selectors to identify groups of related resources. Because selectors are used to query labels, this operation needs to be efficient. To ensure efficient queries, labels are constrained by RFC 1123. RFC 1123, among other constraints, restricts labels to a maximum 63 character length. Thus, labels should be used when you want Kubernetes to group a set of related resources.

Annotations are used for “non-identifying information” i.e., metadata that Kubernetes does not care about. As such, annotation keys and values have no constraints. Thus, if you want to add information for other humans about a given resource, then annotations are a better choice.

The Annotation added to the Ingress resource adds metadata to specify the issuer responsible for requesting certificates. The issuer here will be the same one we have created earlier with the name letsencrypt-prod.


  annotations:
    
    cert-manager.io/cluster-issuer: "letsencrypt-prod"


The other section is tls where the host name that will require https is specified. The secretName also holds the name of the secret that will be created which will store details of the certificate key-pair. i.e Private key and public certificate.


  tls:
  
  - hosts:

    - "tooling.artifactory.egbombah.store"
    
    secretName: "tooling.artifactory.egbombah.store"

Redeploying the newly updated ingress will go through the process as shown below.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/39eba7dc-8f89-42d1-b1b7-bc7d8065fca8)


Once deployed, you can run the following commands to see each resource at each phase.

kubectl get certificaterequest

kubectl get order

kubectl get challenge

kubectl get certificate

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/c2cb2cef-7667-4abe-ab1f-391e1d6cef68)

- At each stage you can run describe on each resource to get more information on what cert-manager is doing.

- If all goes well, running kubectl get certificate,you should see an output like below.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/71489672-d7eb-4fc5-baae-294a3a099c7d)


- Notice the secret name there in the above output. Executing the command

kubectl get secret tooling.artifactory.onyeka.ga -o yaml -n tools

you will see the data with encoded version of both the private key tls.key and the public certificate tls.crt. This is the actual certificate configuration that the ingress controller will use as part of Nginx configuration to terminate TLS/SSL on the ingress.


![](https://github.com/UzonduEgbombah/project-25/assets/137091610/2f813b6a-1127-4b8e-95cb-2b9496a9a638)


- If you now head over to the browser, you should see the padlock sign without warnings of untrusted certificates.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/b3d4d3e1-883a-402e-bcd1-a199f4e2c495)


Finally, one more task for you to do is to ensure that the LoadBalancer created for artifactory is destroyed. If you run a get service kubectl command like below;

kubectl get service -n tools

You will see that the load balancer is still there.

A task for you is to update the helm values file for artifactory, and ensure that the artifactory-artifactory-nginx service uses ClusterIP

Your final output should look like this.

#### Task:
- Input command below 

helm upgrade --install artifactory jfrog/artifactory --set nginx.service.type=ClusterIP,databaseUpgradeReady=true -n tools

Before [Load-balancer present]

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/f69a9695-7088-48e2-89b9-ffb0b6477499)

Updated [Cluster-ip]

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/c1c284f0-7ee9-4827-8afb-618e624b01cf)


Finally, update the ingress to use artifactory-artifactory-nginx as the backend service instead of using artifactory. Remember to update the port number as well.


![](https://github.com/UzonduEgbombah/project-25/assets/137091610/b5f38136-43fd-48f3-926e-8b1fe538fcd7)


- If everything goes well, you will be prompted at login to set the BASE URL. It will pick up the new https address. Simply click next

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/b0ce6e9b-fd43-4bab-b78b-51357c4ad44e)

- Skip the proxy part of the setup.

- Skip repositories creation because you will do this in the next poject.

Then complete the setup.

![](https://github.com/UzonduEgbombah/project-25/assets/137091610/f25968f5-613c-4228-9bd7-5fee68ee9d26)

#### For upgrading and troubleshooting

helm upgrade -i ingress-nginx ingress-nginx/ingress-nginx --set controller.service.type=ClusterIP -n ingress-nginx

helm upgrade -i ingress-nginx ingress-nginx/ingress-nginx --set controller.service.type=LoadBalancer -n ingress-nginx


