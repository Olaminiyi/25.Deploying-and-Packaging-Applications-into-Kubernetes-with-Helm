# Project 25
# Deploying and Packaging Applications into Kubernetes with Helm

In [Project-24](https://github.com/Olaminiyi/24.Setting-up-EKS-with-Terraform-Deploying-Jenkins-Server-using-Helm), we acquired practical skills in using `Helm` to deploy applications on Kubernetes.

Now, in this project, we are focusing on deploying a suite of DevOps tools. Our goal is to confront and understand the typical challenges faced in real-world deployments while mastering effective troubleshooting strategies. We will explore the customization of Helm values files to automate application setups. Throughout the process of deploying different DevOps tools, we'll actively interact with them, comprehending their place in the DevOps lifecycle and how they integrate into the larger ecosystem.

Our primary focus will be on:

- Artifactory
- Ingress Controllers
- Cert-Manager
Then

- Prometheus
- Grafana
- Elasticsearch ELK using [ECK](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-install-helm.html).

**Artifactory** is part of a suit of products from a company called [Jfrog](https://jfrog.com/). Jfrog started out as an artifact repository where software binaries in different formats are stored. Today, Jfrog has transitioned from an artifact repository to a DevOps Platform that includes CI and CD capabilities. This has been achieved by offering more products in which Jfrog Artifactory is part of. Other offerings include

- JFrog Pipelines - a CI-CD product that works well with its Artifactory repository. Think of this product as an alternative to Jenkins.
- JFrog Xray - a security product that can be built-into various steps within a JFrog pipeline. Its job is to scan for security vulnerabilities in the stored artifacts. It is able to scan all dependent code.

### Project Requirement

In this project, the requirement is to use Jfrog Artifactory as a private registry for the organisation's Docker images and Helm charts. This requirement will satisfy part of the company's corporate security policies to never download artifacts directly from the public into production systems. We will eventually have a CI pipeline that initially pulls public docker images and helm charts from the internet, store in artifactory and scan the artifacts for security vulnerabilities before deploying into the corporate infrastructure. Any found vulnerabilities will immediately trigger an action to quarantine such artifacts.

### Deploy Jfrog Artifactory into Kubernetes

First, we provision the kubernetes cluster using eksctl. See [Project-22](https://github.com/Olaminiyi/22.Deploying-application-into-Kubernetes-Cluster).

Create the cluster
```
eksctl create cluster --name ola-eks-tooling2 --region us-west-1 --nodegroup-name worker --node-type t3.medium --nodes 2
```
![alt text](images/25.1.png)

![alt text](images/25.2.png)

![alt text](images/25.3.png)


Create kubeconfig file using awscli and connect to the kubectl.
```
aws eks update-kubeconfig --name ola-eks-tooling2 --region us-west-1
```

Create a namespace tools where all the DevOps tools will be deployed. We will also be deploying jenkins from the previous project in this namespace.
```
kubectl create ns tools
```
![alt text](images/25.4.png)

**Create EBS-CSI Driver for the Cluster**

An `EBS CSI driver` is a crucial component in a `Kubernetes cluster` that utilizes `Amazon Elastic Block Store (EBS`)` for persistent storage. It enables seamless integration between Kubernetes and EBS, allowing for dynamic provisioning, management, and lifecycle control of EBS volumes for containerized applications.

**Here are the key reasons why an EBS CSI driver is essential for a Kubernetes cluster:**

1. Dynamic Provisioning: The EBS CSI driver eliminates the need for manual EBS volume creation and configuration, enabling dynamic provisioning of EBS volumes directly within Kubernetes. This streamlines the storage provisioning process and reduces administrative overhead.

2. Automated Attachment: The EBS CSI driver automatically attaches and detaches EBS volumes to the appropriate Kubernetes nodes based on pod scheduling. This ensures that containers have access to the required storage without manual intervention.

3. Volume Lifecycle Management: The EBS CSI driver manages the entire lifecycle of EBS volumes, including creation, deletion, resizing, and snapshotting. This provides a unified approach to storage management within Kubernetes.

4. Simplified Storage Management: The EBS CSI driver simplifies storage management in Kubernetes by decoupling the storage interface from the Kubernetes controller manager. This allows for more efficient storage management and reduces the complexity of the Kubernetes control plane.

5. Enhanced Storage Flexibility: The EBS CSI driver supports a variety of EBS volume configurations, including different volume types, sizes, and performance options. This provides greater flexibility in tailoring storage to specific application requirements.

6. Integration with Kubernetes Ecosystem: The EBS CSI driver is fully integrated with the Kubernetes ecosystem, including Kubernetes PersistentVolumes, PersistentVolumeClaims, and StorageClasses. This allows for seamless integration with existing Kubernetes storage workflows.

Overall, the **EBS CSI** driver plays a critical role in enabling Kubernetes clusters to effectively leverage EBS for persistent storage. It simplifies storage management, automates volume lifecycle operations, and enhances storage flexibility, making it an indispensable tool for Kubernetes environments.

**Installing EBS CSI Driver**

Run the command to see the pods in the kube-system namespace
```
kubectl get pods -n kube-system
```
![alt text](images/25.5.png)

By running this command, you will observe the presence of **coredns, kube-proxy, and aws-nodes.**

Once the `Ebs-csi driver` is installed, additional nodes related to ebs-csi will appear in the kube-system namespace.

Check the link to setup the EBS CSI add-on

Use this command to check the necessary platform version.
```
aws eks describe-addon-versions --addon-name aws-ebs-csi-driver
```
![alt text](images/25.6.png)

from the above we can see the platform version
```
 "addonVersion": "v1.30.0-eksbuild.1",
                    "architecture": [
                        "amd64",
                        "arm64"
```

You might already have an **AWS IAM OpenID Connect (OIDC)** provider for your cluster. To confirm its existence or establish a new one, check the **OIDC issuer URL** linked to your cluster. An **IAM OIDC** provider is necessary for utilizing IAM roles with service accounts. You can set up an IAM OIDC provider for your cluster using either **eksctl**** or the **AWS Management Console**.

To create an **IAM OIDC** identity provider for your cluster with **eksctl**

Determine the OIDC issuer ID for your cluster.

Retrieve your cluster's **OIDC issuer ID** and store it in a variable.
```
cluster_name=ola-eks-tooling2
```
```
echo $cluster_name
```
```
oidc_id=$(aws eks describe-cluster --name $cluster_name --region us-west-1 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```
```
echo $oidc_id
```
![alt text](images/25.7.png)

Check if there's an **IAM OIDC** provider in your account that matches your cluster's issuer ID.
```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If output is returned, then you already have an IAM OIDC provider for your cluster and you can skip the next step. If no output is returned, then you must create an IAM OIDC provider for your cluster.

In this case, no output was returned.

![alt text](images/25.8.png)

Create an **IAM OIDC**** identity provider for your cluster with the following command
```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --region us-west-1 --approve
```
![alt text](images/25.9.png)

### Configuring a Kubernetes service account to assume an IAM role

**The code below will create Create a file aws-ebs-csi-driver-trust-policy.json that includes the permissions for the AWS services**

```
cat >aws-ebs-csi-driver-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::992382761454:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/${oidc_id}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-1.amazonaws.com/id/${oidc_id}:aud": "sts.amazonaws.com",
          "oidc.eks.us-west-1.amazonaws.com/id/${oidc_id}:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF

```

This trust policy essentially allows the **EBS CSI** driver to assume the role associated with the specified OIDC provider and access AWS resources on behalf of the EBS CSI controller service account

Create the role - **AmazonEKS_EBS_CSI_DriverRole**
```
aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file://"aws-ebs-csi-driver-trust-policy.json"
```
Attach a policy. AWS maintains an AWS managed policy or you can create your own custom policy. Attach the AWS managed policy to the role.
```
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```
![alt text](images/25.10.png)

![alt text](<images/Screenshot 2024-06-03 at 12.14.47.png>)

To add the Amazon **EBS CSI add-on** using the **AWS CLI**

Run the following command.
```
aws eks create-addon --cluster-name $cluster_name --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::992382761454:role/AmazonEKS_EBS_CSI_DriverRole --region us-west-1
```
![alt text](images/25.11.png)

Execute the command, you will find the ebs-csi driver related pods available.
```
kubectl get pods -n kube-system
```
![alt text](images/25.12.png)

### Installing the tools in kubernetes

Install artifactory in the namespace tools

Search for an official helm chart for Artifactory on [Artifact Hub](https://artifacthub.io/).

![alt text](images/25.15.png)

**Click** on install to display the commands for installation. on the right hand side of the page

![alt text](images/25.16.png)
![alt text](images/25.18.png)

Add the repo
```
helm repo add jfrog https://charts.jfrog.io
```
Update the helm repo index on my local machine/laptop
```
helm repo update
```
Install artifactory in the namespace tools
```
helm upgrade --install artifactory jfrog/artifactory --version 107.71.4 -n tools
```
![alt text](images/25.17.png)

We opted for the **upgrade --install** flag over **helm install** artifactory jfrog/artifactory for enhanced best practices, especially in **CI pipeline development for helm deployments**. This approach guarantees that helm performs an upgrade if an installation exists. In the absence of an existing installation, it conducts the initial install. This strategy assures a fail-safe command; it intelligently discerns whether an upgrade or a fresh installation is needed, preventing failures.

To see the various versions; scrow down and check the right side of the page and click on see all link

![alt text](images/25.19.png)
![alt text](images/25.20.png)

**Getting the Artifactory URL**

The artifactory helm chart comes bundled with the `Artifactory software`, a `PostgreSQL database` and an `Nginx proxy` which it uses to configure routes to the different capabilities of Artifactory. Getting the pods after some time, you should see something like the below.

> [!NOTE]
> After installing artifactory with helm, I faced an error due to the artifactory pods been in the pending mode.
> After investigation using the `describe pod` method, I was getting the error message below
> **"running PreBind plugin "VolumeBinding": binding volumes: context deadline exce"**
> The issue was a result of the `EBS_CSI_DriverRole` that was not created properly. The `oidc_id` was not passing the actual value to the policy required for the creation of the `EBS_CSI_DriverRole`. The interpolation method to pass the value of `oidc_id` was changed from `$oidc_id` to `{oidc_id}`. This resolved the issue

Renamed the project directory to a shorter name to make it more comprehensible and to follow naming convention standard. **You may not need to do this if you didn't named your project directory with a long name like mine**
```
mv 25.Deploying and Packaging Applications into Kubernetes with Helm  Deploy
```

```
kubectl get pods -n tools -w
```
![alt text](images/25.21.png)

Each of the deployed application have their respective services. This is how you will be able to reach either of them.

Notice that, the `Nginx Proxy` has been configured to use the service type of `LoadBalancer`. Therefore, to reach Artifactory, we will need to go through the `Nginx proxy's service`. Which happens to be a load balancer created in the cloud provider.
```
kubectl get svc -n tools
```
![alt text](images/25.22.png)


**Accessing the artifactory using the Load balancer URL***

![alt text](images/25.23.png)

Login using default

**username: admin**

**password: password**

![alt text](images/25.24.png)

**How the Nginx URL for Artifactory is configured in Kubernetes**

How did **Helm** configure the **URL** in kubernetes?

Helm uses the values.yaml file to set every single configuration that the chart has the capability to configure. The best place to get started with an off the shelve chart from artifacthub.io is to get familiar with the DEFAULT VALUES section on the right side of the Artifact hub's website.

![alt text](images/25.25.png)


**Explore key and value pairs within the system.**

For instance, entering "nginx" into the search bar will display all the configured options for the nginx proxy. Choosing **"nginx.enabled"**** from the list will promptly navigate you to the corresponding configuration in the YAML file.

![alt text](images/25.26.png)

Search for **nginx.service**** and choose **nginx.service.type**. This will display the configured Kubernetes service type for **Nginx**. By default, it appears as **LoadBalancer**.

![alt text](images/25.27.png)

![alt text](images/25.28.png)

To work directly with the values.yaml file, you can download the file locally by clicking on the download icon.

![alt text](images/25.29.png)

### Ingress controller

Configuring applications in Kubernetes to be externally accessible often begins with setting the service type to a Load Balancer. However, this can lead to escalating expenses and complex management as the number of applications grows, resulting in a multitude of provisioned load balancers.

An optimal solution lies in leveraging Kubernetes Ingress instead, as detailed in Kubernetes Ingress documentation. Yet, this transition requires deploying an Ingress Controller.

One major advantage of employing an Ingress controller is its **ability to utilize a single load balancer across various deployed applications**. This consolidation allows for the reuse of the load balancer by services like Artifactory and other tools. Consequently, it significantly reduces cloud expenditure and minimizes the overhead associated with managing multiple load balancers, a topic we'll delve deeper into shortly.

For now, we will leave artifactory, move on to the next phase of configuration (Ingress, DNS(Route53) and Cert Manager), and then return to Artifactory to complete the setup so that it can serve as a private docker registry and repository for private helm charts.

### Deploying Ingress Controller and managing Ingress Resources

An ingress in Kubernetes is an API object responsible for overseeing external access to services within the cluster. It handles tasks like load balancing, SSL termination, and name-based virtual hosting. Essentially, an Ingress facilitates the exposure of HTTP and HTTPS routes from outside the cluster to services within cluster. Traffic direction is managed by rules set within the Ingress resource.

To illustrate, here is a straightforward example where an Ingress directs all its traffic to a single Service

![alt text](images/ingress.png)

An ingress resource for Artifactory would look like this

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:
  - host: "tooling.artifactory.olami.uk"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082

```

- An Ingress needs **apiVersion, kind, metadata and spec fields**
- The name of an Ingress object must be a valid DNS subdomain name
- Ingress frequently uses annotations to configure some options depending on the Ingress controller.
- Different Ingress controllers support different annotations. Therefore it is important to be up to date with the ingress controller's specific documentation to know what annotations are supported.
- It is recommended to always specify the ingress class name with the spec ingressClassName: nginx. This is how the Ingress controller is selected, especially when there are multiple configured ingress controllers in the cluster.
- The domain olami.uk should be replaced with your own domain which has already been purchased from domain providers and configured in AWS Route53.
- Artifactory in the backend field is the name of the artifactory service we already have running.

If you attempt to apply the specified YAML configuration for the ingress resource without an ingress controller, it won't function. For the Ingress resource to operate, the cluster must have an active ingress controller. Unlike various controllers running as part of the kube-controller-manager—like the Node Controller, Replica Controller, Deployment Controller, Job Controller, or Cloud Controller—Ingress controllers don't initiate automatically with the cluster. Kubernetes officially supports and maintains AWS, GCE, and NGINX ingress controllers. However, numerous other 3rd-party Ingress controllers exist, offering similar functionalities alongside their distinct features. Among these, the officially supported ones include:

- [AKS Application Gateway Ingress Controller (Microsoft Azure)](https://learn.microsoft.com/en-gb/azure/application-gateway/tutorial-ingress-controller-add-on-existing)
- [Istio](https://istio.io/latest/docs/tasks/traffic-management/ingress/kubernetes-ingress/)
- [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [Ambassador](https://www.getambassador.io/)
- [HA Proxy Ingress](https://haproxy-ingress.github.io/)
- [Kong](https://docs.konghq.com/kubernetes-ingress-controller/latest/)
- [Gloo](https://docs.solo.io/gloo-edge/latest/)


While there are more 3rd-party Ingress controllers available, the aforementioned ones are currently backed and maintained by Kubernetes. A [comparison matrix](https://kubevious.io/blog/post/comparing-top-ingress-controllers-for-kubernetes#comparison-matrix) of these controllers can assist in understanding their unique traits, aiding businesses in choosing the right fit for their requirements. In a cluster, it's feasible to deploy multiple ingress controllers, thanks to the essence of an ingress class. By specifying the `spec ingressClassName` field on the ingress object, the appropriate ingress controller will be utilized by the ingress resource.

### Deploy Nginx Ingress Controller

We will deploy and use the Nginx Ingress Controller. It is always the default choice when starting with Kubernetes projects. It is reliable and easy to use. Since this controller is maintained by Kubernetes, there is an official guide the installation process. Hence, we wont be using artifacthub.io here. Even though you can still find ready to go charts there, it just makes sense to always use the [official guide](https://kubernetes.github.io/ingress-nginx/deploy/) in this scenario.

Using the Helm approach, according to the official guide;

- Install Nginx Ingress Controller in the ingress-nginx namespace

```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

```
This command is idempotent. It installs the ingress controller if it is not already present. However, if the ingress controller is already installed, it will perform an upgrade instead.
```
kubectl get pods --namespace=ingress-nginx
```
The following command will wait for the ingress controller pod to be up, running, and ready
```
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

**OR**

We can also install the ingress-nginx using the same approach used in installing jenkins and artifactory.

**Create a name space ingress-nginx**
```
kubectl create ns ingress-nginx
```

Search for an official helm chart for ingress-nginx on [Artifact Hub](https://artifacthub.io/).
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
Update repo
```
helm repo update
```
Install ingress-nginx
```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --version 4.8.3 -n ingress-nginx
```
![alt text](images/25.30.png)


Get pods in the ingress-nginx namespace
```
kubectl get pods -n ingress-nginx -w
```
![alt text](images/25.31.png)

Check to see the created load balancer in AWS.
```
kubectl get svc -n ingress-nginx
```
![alt text](images/25.32.png)

The ingress-nginx-controller service that was created is of the type LoadBalancer. That will be the load balancer to be used by all applications which require external access, and is using this ingress controller. If you go ahead to AWS console, copy the address in the EXTERNAL-IP column, and search for the loadbalancer, you will see an output like below.

![alt text](images/25.33.png)

If you go ahead to AWS console, copy the address in the EXTERNAL-IP column, and search for the loadbalancer, you will see an output like below

![alt text](images/25.34.png)

**Deploy Artifactory Ingress**

Now, it is time to configure the ingress so that we can route traffic to the Artifactory internal service, through the ingress controller's load balancer.

![alt text](images/image2.png)

Notice the section with the configuration that selects the `ingress controller` using the `ingressClassName`
```
cat <<EOF > artifactory-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:
  - host: "tooling.artifactory.olami.uk"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082
EOF
```

Create the ingress resource in the tools namespace
```
kubectl apply -f artifactory-ingress.yaml -n tools
```
To get the IngressClass that identifies this ingress controller in the above, Run:
```
kubectl get ingressclass -n ingress-nginx
```

Get the ingress resource
```
$ kubectl get ingress -n tools
```
![alt text](images/25.36.png)


**Note:**

**CLASS** - The nginx controller class name nginx

**HOSTS** - The hostname to be used in the browser tooling.artifactory.olami.uk

**ADDRESS** - The loadbalancer address that was created by the ingress controller

**Configure DNS**

When accessing the tool, sharing the lengthy load balancer address poses significant inconvenience. The ideal solution involves creating a DNS record that's easily readable by humans and capable of directing requests to the balancer. This exact configuration is set within the ingress object as host: "tooling.artifactory.olami.uk". However, without a corresponding DNS record, this host address cannot reach the load balancer.

The "olami.uk" portion of the domain represents the configured HOSTED ZONE in AWS. To enable this functionality, it's necessary to set up the Hosted Zone in the AWS console or include it as part of your infrastructure using tools like Terraform.

Create hosted zone olami.uk

You must have purchased a domain name from a domain provider and configured the nameservers.

![alt text](images/25.37.png)

![alt text](images/25.38.png)

![alt text](images/25.39.png)

![alt text](images/25.40.png)

**Create Route53 record**

To establish a Route53 record, navigate to the hosted zone where essential DNS records are managed. For Artifactory, let's configure a record directing to the load balancer of the ingress controller. You have two choices: utilize either the CNAME or AWS Alias method.

If opting for the CNAME Method,

- Choose the desired HOSTED ZONE.
- Put the Loadbalancer address in the `value- text box`
- Click on the "Create Record" button to proceed.



![alt text](images/25.41.png)

Please verify the DNS record's successful propagation. Go to [DNS checker](https://dnschecker.org/) and choose CNAME to check the record. Make sure there are green ticks next to each location on the left-hand side. Please note that it may take some time for the changes to propagate.

![alt text](images/25.42.png)

We can also check this using the command
```
nslookup -type=ns tooling.artifactory.olami.uk
```
![alt text](images/25.43.png)


**AWS Alias Method**

In the create record section, type in the record name, and toggle the alias button to enable an alias. An alias is of A DNS record type which basically routes directly to the load balancer. In the choose endpoint bar, select Alias to Application and Classic Load Balancer.

**Accessing the application from the browser**

we now have an application running in Kubernetes that is also accessible externally. That means if you navigate to https://tooling.artifactory.olami.uk it should load up the artifactory application.

When accessing the application via the HTTPS protocol in Chrome, you might see a message stating that the site is reachable but insecure. This happens when the site doesn't have a trusted TLS/SSL certificate, or it lacks one entirely.

![alt text](images/25.44.png)

We can access this using Edge browser or Safari

![alt text](images/25.45.png)

From the above, we can see that the ingress-nginx Controller does configure a default TLS/SSL certificate. But it is not trusted because it is a self signed certificate that browsers are not aware of.

To view the certificate, click on the "Not Secure" section and then select "Your connection to this site isn't secure"

![alt text](images/25.46.png)

The Nginx Ingress Controller sets up a default TLS/SSL certificate. However, it is self-signed and not recognized by browsers, which means it is not trusted. To verify this, click on the "Not Secure" label on the browser.

**Explore Artifactory Web UI**

Get the default username and password - Run a helm command to output the same message after the initial install
```
helm test artifactory -n tools
```
![alt text](images/25.47.png)

Insert the username and password to load the Get Started page

Reset the admin password

Activate the Artifactory License. You will need to purchase a license to use Artifactory enterprise features.

![alt text](images/25.48.png)

For learning purposes, you can apply for a free trial license. Simply fill the form [here](https://jfrog.com/start-free/) and a license key will be delivered to your email in few minutes.

N/B: Make sure to check the box "schedule a technical demo"

![alt text](images/25.49.png)

![alt text](images/25.50.png)

![alt text](images/25.51.png)

![alt text](images/25.52.png)

Click on **next**

Skip Proxy settings and creating the repo for now.

![alt text](images/25.53.png)

![alt text](images/25.54.png)

![alt text](images/25.55.png)

click **Finish**

![alt text](images/25.56.png)

Next, its time to fix the TLS/SSL configuration so that we will have a trusted HTTPS URL

### Deploying Cert-Manager and managing TLS/SSL for Ingress

Transport Layer Security (TLS), the successor of the now-deprecated Secure Sockets Layer (SSL), is a cryptographic protocol designed to provide communications security over a computer network. The TLS protocol aims primarily to provide cryptography, including privacy (confidentiality), integrity, and authenticity through the use of certificates, between two or more communicating computer applications. The certificates required to implement TLS must be issued by a trusted Certificate Authority (CA). To see the list of trusted root Certification Authorities (CA) and their certificates used by Google Chrome, you need to use the Certificate Manager built inside Google Chrome as shown below:

Open the settings section of google chrome

Search for security and click on `Security` - Safe Browsing (protection from dangerous sites) and other security settings

Select **safe browsing(protection from dangerous site) and other security setting**

![alt text](images/25.57.png)

Select **Manage Certificates**

![alt text](images/25.58.png)

View the installed certificates in your browser

![alt text](images/25.59.png)


### Certificate Management in Kubernetes

Streamlining the acquisition and management of trusted certificates from dynamic certificate authorities is a challenging task. It involves overseeing certificate requests, issuance, expiration tracking, and application-specific certificate management, which can incur significant administrative overhead. This often requires the creation of intricate scripts or programs to handle these complexities.

[Cert-Manager](https://cert-manager.io/) is a lifesaver in simplifying these processes. Within Kubernetes clusters, Cert-Manager introduces certificates and certificate issuers as resource types. It streamlines the acquisition, renewal, and utilization of certificates same approach the Ingress Controllers facilitate the creation of Ingress resources within the cluster.

Cert-Manager empowers administrators by enabling the creation of certificate resources and additional resources essential for seamless certificate management. It supports certificate issuance from various sources like Let's Encrypt, HashiCorp Vault, Venafi, and private PKIs. The resulting certificates are stored as Kubernetes secrets, housing both the private key and the public certificate for easy access and utilization.

![alt text](images/image3.png)

In this Project, we will use **Let's Encrypt** with cert-manager.

The certificates issued by Let's Encrypt will work with most browsers because the root certificate that validates all it's certificates is called “ISRG Root X1” which is already trusted by most browsers and servers. You will find ISRG Root X1 in the list of certificates already installed in your browser.

Cert-manager will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry.

**Cert-Manager high Level Architecture**

**Cert-manager** works by having administrators create a resource in kubernetes called certificate issuer which will be configured to work with supported sources of certificates. This issuer can either be scoped globally in the cluster or only local to the namespace it is deployed to. Whenever it is time to create a certificate for a specific host or website address, the process follows the pattern seen in the image below.

![alt text](images/image4.png)

**Deploying Cert-manager**

Lets Deploy cert-manager helm chart in Artifact Hub, follow the installation guide and deploy into Kubernetes

Create a namespace cert-manager
```
kubectl create ns cert-manager
```
Before installing the chart, you must first install the cert-manager CustomResourceDefinition resources. This is performed in a separate step to allow you to easily uninstall and reinstall cert-manager without deleting your installed custom resources.
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml -n cert-manager
```
Add the Jetstack helm repo
```
helm repo add jetstack https://charts.jetstack.io
```
Install the cert-manager helm chart
```
helm install cert-manager jetstack/cert-manager --version v1.14.5 --namespace cert-manager
```
![alt text](images/25.60.png)

**Certificate Issuer**

In order to begin issuing certificates, you will need to set up a ClusterIssuer or Issuer resource.

Create an Issuer. We will use a Cluster Issuer so that it can be scoped globally. Assuming that we will be using olami.uk domain. Simply update this yaml file and deploy with kubectl. In the section that follows, we will break down each part of the file.

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  namespace: "cert-manager"
  name: "letsencrypt-prod"
spec:
  acme:
    server: "https://acme-v02.api.letsencrypt.org/directory"
    email: "infradev@oldcowboyshop.com"
    privateKeySecretRef:
      name: "letsencrypt-prod"
    solvers:
    - selector:
        dnsZones:
          - "olami.uk"
      dns01:
        route53:
          region: "us-west-1"
          hostedZoneID: "Z2CD4NTR2FDPZ"
```

The initial section shows the Kubernetes configuration, specifying the apiVersion, Kind and metadata. In this context, the Kind refers to a ClusterIssuer, indicating its global scope.

In the spec section, an ACME - Automated Certificate Management Environment issuer type is specified here. When you create a new ACME Issuer, cert-manager will generate a private key which is used to identify you with the ACME server. Certificates issued by public ACME servers are typically trusted by client's computers by default. This means that, for example, visiting a website that is backed by an ACME certificate issued for that URL, will be trusted by default by most client's web browsers. ACME certificates are typically free. Let’s Encrypt uses the ACME protocol to verify that you control a given domain name and to issue you a certificate. You can either use the let's encrypt Production server address https://acme-v02.api.letsencrypt.org/directory which can be used for all production websites. Or it can be replaced with the staging URL https://acme-staging-v02.api.letsencrypt.org/directory for all Non-Production sites.

The privateKeySecretRef has configuration for the private key name you prefer to use to store the ACME account private key. This can be anything you specify, for example letsencrypt-prod.

This section is part of the spec that configures solvers which determines the domain address that the issued certificate will be registered with. dns01 is one of the different challenges that cert-manager uses to verify domain ownership. Read more on DNS01 Challenge here. With the DNS01 configuration, you will need to specify the Route53 DNS Hosted Zone ID and region. Since we are using EKS in AWS, the IAM permission of the worker nodes will be used to access Route53. Therefore if appropriate permissions is not set for EKS worker nodes, it is possible that certificate challenge with Route53 will fail, hence certificates will not get issued.

The next section under the spec that configures solvers which determines the domain address that the issued certificate will be registered with. dns01 is one of the different challenges that cert-manager uses to verify domain ownership. Read more on [DNS01 Challenge here](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge). With the __ DNS01__ configuration, you will need to specify the Route53 DNS Hosted Zone ID and region. Since we are using EKS in AWS, the IAM permission of the worker nodes will be used to access Route53. Therefore if appropriate permissions is not set for EKS worker nodes, it is possible that certificate challenge with Route53 will fail, hence certificates will not get issued. The other possible option is the HTTP01 challenge, but we won't be using that.

To get the Hosted Zone ID
```
aws route53 list-hosted-zones
```

![alt text](images/25.61.png)


Update the yaml file with the correct Hosted Zone ID.

```
cat <<EOF > cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  namespace: "cert-manager"
  name: "letsencrypt-prod"
spec:
  acme:
    server: "https://acme-v02.api.letsencrypt.org/directory"
    email: "infradev@oldcowboyshop.com"
    privateKeySecretRef:
      name: "letsencrypt-prod"
    solvers:
    - selector:
        dnsZones:
          - "olami.uk"
      dns01:
        route53:
          region: "us-west-1"
          hostedZoneID: "Z07846632EVOFYUZSWK8F"
EOF
```

Deploy with kubectl in the cert-manager namespace
```
kubectl apply -f cluster-issuer.yaml -n cert-manager
```

![alt text](images/25.62.png)

```
kubectl get pods -n cert-manager
```

![alt text](images/25.63.png)

With the ClusterIssuer properly configured, it is now time to start getting certificates issued.

**Configuring Ingress for TLS**

To ensure that every created ingress also has TLS configured, we will need to update the ingress manifest with TLS specific configurations.

```
cat <<EOF > artifactory-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: nginx
  name: artifactory
spec:
  rules:
  - host: "tooling.artifactory.olami.uk"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082
  tls:
  - hosts:
    - "tooling.artifactory.olami.uk"
    secretName: "tooling.artifactory.olami.uk"
EOF
```

Create the updated artifactory-ingress.yaml
```
kubectl apply -f artifactory-ingress.yaml -n tools
```
![alt text](images/25.64.png)

After updating the ingress above, the artifactory will be inaccessible through the browser.

![alt text](images/25.65.png)

The most significant updates to the ingress definition is the annotations and tls sections.

Annotations are used similar to labels in kubernetes. They are ways to attach metadata to objects.

**Difference between Annotations and Label**

Annotations and labels serve distinct roles in Kubernetes resource management.

Labels function as identifiers for resource grouping when used alongside selectors. To ensure efficient querying, labels adhere to [RFC 1123](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#:~:text=RFC%201123%20Label%20Names&text=in%20RFC%201123.-,This%20means%20the%20name%20must%3A,start%20with%20an%20alphanumeric%20character) constraints, limiting their length to 63 characters. Therefore, utilizing labels is ideal for Kubernetes when organizing related resources into sets.

On the other hand, annotations cater to "non-identifying information" or metadata that Kubernetes doesn't rely on. Unlike labels, there are no constraints imposed on annotation keys and values. Consequently, if the goal is to furnish additional information for human comprehension regarding a specific resource, annotations offer a more suitable choice.

The Annotation added to the Ingress resource adds metadata to specify the issuer responsible for requesting certificates. The issuer here will be the same one we have created earlier with the name letsencrypt-prod

```
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```
The other section is tls where the host name that will require https is specified. The secretName also holds the name of the secret that will be created which will store details of the certificate key-pair. i.e Private key and public certificate.

```
  tls:
  - hosts:
    - "tooling.artifactory.olami.uk"
    secretName: "tooling.artifactory.olami.uk"
```
Redeploying the newly updated ingress will go through the process as shown below.

![alt text](images/image5.png)

commands to see each resource at each phase.
```
kubectl get certificaterequest -n tools
```
```
kubectl get order -n tools
```
```
kubectl get challenge -n tools
```
```
kubectl get certificate -n tools
```

> [!IMPORTANT]
> Problem encoutered
> After applying the above command, i ran into some issues, my certicate remains in pending state.
> ![alt text](images/25.66.png)

**During investigation on the challenge resource, I noticed a permission issue**
```
kubectl describe challenge tooling.artifactory.olami.uk-1-3686652334-1442975272 -n tools
```

![alt text](images/25.67.png)

> [!NOTE]
> This means that there is an issue with presenting a challenge due to a permissions error related to Route 53 in AWS. The error indicates that the IAM role being assumed (eksctl-ola-eks-tooling2-nodegroup--NodeInstanceRole-znPcc1f2R8go/i-09dfdcc872addf2fa ) does not have the necessary permissions to perform the route53:ChangeResourceRecordSets action on the specified hosted zone (Z07846632EVOFYUZSWK8F).


**Resolution**

To resolve the permissions issue for the IAM role eksctl-ola-eks-tooling2-nodegroup--NodeInstanceRole-znPcc1f2R8go/i-09dfdcc872addf2fa and allow it to perform the route53:ChangeResourceRecordSets action on the hosted zone (Z07846632EVOFYUZSWK8F).

Identify the IAM role assumed by your EKS cluster nodes - eksctl-ola-eks-tooling2-nodegroup--NodeInstanceRole-znPcc1f2R8go

![alt text](images/25.68.png)


Update the IAM policy linked to this role by adding the required permissions for Route 53. Ensure that you authorize the route53:ChangeResourceRecordSets and route53:GetChange actions for the designated hosted zone arn:aws:route53:::hostedzone/Z07846632EVOFYUZSWK8F.

![alt text](images/25.69.png)

You can see that the `route53:ChangeResourceRecordSets` is not included in the permission policy above.

You can also get the attached policies using this command
```
 aws iam list-attached-role-policies --role-name eksctl-ola-eks-tooling2-nodegroup--NodeInstanceRole-znPcc1f2R8go
```
Create the IAM policy to be added to the role policy

```
cat <<EOF > ChangeResourceRecordSets.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement",
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets",
                "route53:GetChange"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/Z07846632EVOFYUZSWK8F",
                "arn:aws:route53:::change/*"
            ]
        }
    ]
}
EOF
```
![alt text](images/25.70.png)

Apply the updated IAM policy to the IAM role. You can do this through the AWS Management Console or by using the AWS CLI
```
aws iam create-policy --policy-name ChangeResourceRecordSets --policy-document file://ChangeResourceRecordSets.json
```

After running the create-policy command, the output will include the Amazon Resource Name (ARN) of the created policy.

Then attach the policy to the role

```
aws iam attach-role-policy --role-name eksctl-ola-eks-tooling2-nodegroup--NodeInstanceRole-znPcc1f2R8go --policy-arn arn:aws:iam::992382761454:policy/ChangeResourceRecordSets
```
![alt text](images/25.71.png)

On the policies, we will see the newly created policy

![alt text](images/25.72.png)

You can verify that the policy has been attached by running
```
aws iam list-attached-role-policies --role-name eksctl-ola-eks-tooling2-nodegroup--NodeInstanceRole-znPcc1f2R8go 
```

![alt text](images/25.73.png)


Restart the pod in the `cert-manager` namespace by deleting it, allowing it to be automatically recreated.

```
kubectl get pods -n cert-manager
```
```
kubectl delete pod cert-manager-796cbd6574-9p5vn -n cert-manager
```
```
kubectl get pods -n cert-manager
```

![alt text](images/25.74.png)


Then check
```
kubectl get certificaterequest -n tools
```
```
kubectl get order -n tools
```
```
kubectl get certificate -n tools
```
```
kubectl get challenge -n tools
```
![alt text](images/25.75.png)

In the screenshot above, you will see that the challenges have been successfully resolved and are no longer present because a certificate has been obtained.

Run
```
kubectl get secret tooling.artifactory.olami.uk -o yaml -n tools
```
![alt text](images/25.76.png)

Refresh the browser, you will find that the site is now secure.

![alt text](images/25.77.png)

Finally, one more task for you to do is to ensure that the LoadBalancer created for artifactory is destroyed. If you run a get service kubectl command like below
```
kubectl get svc -n tools
```
![alt text](images/25.78.png)

You will notice that the load balancer remains intact.

Modify the Helm values file for Artifactory and verify that the artifactory-artifactory-nginx service is set to use ClusterIP.
```
helm show values jfrog/artifactory
```
Redirect the values to a file
```
helm show values jfrog/artifactory > values.yaml
```
```
ls
```

![alt text](images/25.79.png)

```
cat values.yaml
```

![alt text](images/25.80.png)

**Replace the LoadBalancer created for artifactory with ClusterIP**

```
helm upgrade artifactory jfrog/artifactory --set nginx.service.type=ClusterIP,databaseUpgradeReady=true -n tools
```
**nginx.service.type=ClusterIP:** This parameter configures the Nginx service to use a ClusterIP, which is the default service type for internal services in a Kubernetes cluster. This type of service is only accessible from within the cluster. databaseUpgradeReady=true: This parameter is a flag indicating that the database upgrade is ready. This could be part of a process where you are informing Artifactory that the database upgrade is complete, and therefore the application can proceed with any related tasks.

![alt text](images/25.81.png)

Finally, update the ingress to use `artifactory-artifactory-nginx` as the backend service
```
cat <<EOF > artifactory-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: nginx
  name: artifactory
spec:
  rules:
  - host: "tooling.artifactory.olami.uk"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory-artifactory-nginx
            port:
              number: 80
  tls:
  - hosts:
    - "tooling.artifactory.olami.uk"
    secretName: "tooling.artifactory.olami.uk"
EOF
```

Apply
```
kubectl apply -f artifactory-ingress.yaml -n tools
```

If everything goes well, you will be prompted at login to set the BASE URL. It will pick up the new https address. Simply click next

Skip the proxy part of the setup.

Skip repositories creation because we will configure all of this in the next project

Then complete the setup.

![alt text](images/25.82.png)


Congratulations! for completing Project 25

In the next project, you will experience;

- Configuring private container registry using Artifactory
- Configuring private helm repository using Artifactory
- Configuring Custom Helm charts for the tooling app
- Implementing CICD for databas schemas using.







