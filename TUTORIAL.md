# Intro

The idea is to use multiple cloud providers to deploy your application.  
Many large companies use this approach because it solves real technical and business problems.  

But managing multiple clouds is genuinely complex.  
You're dealing with different APIs, different security models, different networking configurations, completely different ways of doing things.  

What about your cloud bill?  
It can literally double just from the operational overhead of managing an infrastructure that is scattered across multiple clouds.  

In this tutorial, we'll learn:
- how multi-cloud setups work in practice
- which problems we'll be facing when we start using the multi-cloud approach
- how to solve these challenges conceptually
- how to deploy a containerized application across AWS, Azure and Google Cloud (practical demo)

# Realistic scenario of multi-cloud

We want to run our main microservices and APIs on AWS using EC2 and EKS, because our CI/CD pipelines, monitoring, networking setup is already 
built around AWS and our team has expertise in it.  

For analytics, we want to stream our application events from AWS into GCP, and store them in BigQuery, where our product and data teams can run 
simple SQL queries over several years of data of user clicks and billing to understand user behavior.  
And they will then create dashboards in tools like Looker Studio to visualize all that data.  

For Machine Learning (ML) features, our backend will call Azure AI services: for example, ready-made language and image models that can read documents 
or check content. Because those pre-built models already solve our problem and save us the effort of training and running our own ML systems.  

And on top of that, even though we run our main application on AWS, we also want to ensure fault tolerance by replicating our app on other cloud platforms.  
We want to run our workload on AWS, GCP and Azure.  

# 2 Multi-Cloud implementation patterns (and their challenges)

## 1. One home cloud + specific services

You run your application primarily in one cloud and consume specific services from others.  
Meaning your main cloud provider does not host your entire application, only most of it.  

This means your application code needs different SDKs, one for each cloud provider.  
Each SDK has different patterns, different authentication methods, different error handling.  

This has two direct consequences:
1) your code becomes more complex because you're integrating multiple cloud platforms in your application
2) in addition to this complexity, you need to manage different authentication methods and as many credentials

### How do you handle networking between clouds?  

For example, when your application running on AWS calls Google Cloud BigQuery service, how does that request goes over the network?  
- is it calling it over the public internet? Which is slow and expensive
- or do you set up a private connection between your AWS and Google Cloud? which requires some complex network configuration

## 2. Full application in multiple clouds

You run copies of your entire application in multiple clouds.  
Advantages: resilience and fault tolerance  

If one cloud provider has a major outage, your application can fail over to another cloud and keep serving users without having any downtime.  

What this means in practice is that you're going to need compute infrastructure in both clouds:  
- For example, if you're using K8s, you'll need EKS cluster on AWS and GKE cluster on Google Cloud.  
- For your VMs, you'll need EC2 instances on AWS, and Compute Engine instances on GCP.  

You'll also need to configure networking in both clouds:  
- You'll need VPCs on AWS with subnets, AWS security groups, AWS load balancers
- You'll also need VPCs on Google Cloud, with their own subnets, GCP firewall rules, GCP load balancers

Finally, we have identity and access management (IAM): defining permissions and how different services can be accessed securely:
- on AWS, you'll need to set up IAM roles and policies
- on GCP, you'll have to set up service accounts and IAM permissions

If you're using **Terraform**, you'll need to have one configuration for your AWS infrastructure and one for your GCP infrastructure.  

And most importantly, you'll need **global traffic management**.  
You'll need a way to route user traffic to the right cloud:
- maybe use DNS-based failover: if one cloud is down, DNS automatically routes traffic to another cloud
- maybe use a global load balancer that sits in front of both clouds, which adds another layer of complexity

# Abstraction Layer

From the two patterns described in the previous section, the best solution is to manage two different cloud providers, such as AWS and GCP, and to have two separate Terraform codes, and so on.  

But given the complexity induced by this solution, it would be nice to have an abstraction layer that simplifies things a little bit...  
An asbtraction layer is some tool that hides the underlying complexity, like Docker abstracts away the need for OS-specific configuration when it comes to deploying applications.  

To remove some of the complexity that is tied to multi-cloud architecture, we need a solution that prevents our application from having to deal with Cloud-specific APIs.  
We need an abstraction layer that our application can communicate with, and that abstraction layer would handle the Cloud-specific details for us.  

That's exactly where a tool called "**Control Plane**" comes in.  

# How does Control Plane implement this abstraction?

https://docs.controlplane.com/whatis  

Control Plane is a multi-cloud management platform that lets teams provision, manage, and govern infrastructure across providers like AWS, Azure, and GCP from a single interface.  
It uses a "Workspace" model to abstract underlying clouds, enabling policy enforcement, cost tracking, and secure access without vendor lock-in.  

It sits on top of AWS, GCP, Azure, or any other cloud provider (even works with on-premises infrastructure).  

## 1. Universal Cloud Identity

This is Control Plane's solution for managing credentials for not only different cloud providers, but also for different services of those cloud providers.  
This allows your microservices to not have cloud-specific credentials.  
They don't have AWS keys or GCP service account files, instead they authenticate with control plane using a **universal identity**.

### Example scenario

Let's say your application needs to read a file from AWS S3.  
Normally, your app would need AWS credentials, such as an access key and secret key, you would need to manage these credentials, rotate them, and keep them secure.  

With Control Plane, your application has a universal cloud identity (UCI).  
When it makes a request to read from S3, it authenticates using its UCI.  
Control Plane receives this request, translates the UCI into temporary AWS credentials, makes the request to S3 and returns the response to your application.  

Your app doesn't even know that this translation happened. From its perspective, it authenticated once and got the data.  
And the same application using the same Control Plane identity can now access other services from other cloud providers.  
Your app doesn't have to embed cloud-specific secrets. Control Plane maps your app's UCI to the right cloud credentials behind the scenes.  

## 2. Service Composition across Clouds

With Control Plane, you can mix & match services from different Clouds in the same application.  

### Example scenario

Let's say you're building an e-commerce application.  
You decide to run the front end on AWS because you're already using AWS CloudFront and CDN.  
But you want to run the recommendation engine on Google Cloud because you're using BigQuery to analyze customer behavior and train your models.  
And you store product images on Azure blob storage because you negotiated a good price with Microsoft.  

These services need to all work together:
- The frontend needs to call the recommendation engine
- The recommendation engine needs to access product images
- and so on...

With Control Plane, you deploy all these services to one single platform.  
And Control Plane handles the networking between clouds automatically, which removes a lot of complexity for Cloud architects and DevOps engineers!  

## 3. Pre-integrated infrastructure stack

Have you ever set up a K8s cluster and configured all these third-party services to make it production-ready?  
Which implies installing Istio, Vault, Prometheus, Grafana, cert-manager, etc.

Control Plane has integrated all those capabilities into their platform.  
So that, when you provision a Kubernetes cluster, you're not starting from bare Kubernetes, all services quoted above come pre-integrated and configured.  

When you deploy an application to the Control Plane platform: 
- observability and metrics collection just works out of the box,
- logs are automatically aggregated,
- distributed tracing is automatically configured,
- and you get dashboards showing everything that's happening in your application.

Similarly, security policies just work: TLS encryption between services is automatic, and the same goes for certificate management.  
You just define which services can talk to which other services, and Control Plane (CP) can enforce this using the built-in service mesh.  

## 4. Optimizing Cloud Costs

And finally, another huge benefit when using multi-cloud is the **cost optimization**.  
Because, cloud costs, even on a single cloud platform, can get very complex, very fast.  

CP actually gives you very precise control over resource allocation.  
So, instead of saying "I need a T3.large instance" and then that instance runs all the time (whether you're using it or not, or using just 50% of it), 
you basically say "my application needs 2 CPU cores and 4GB of memory", and CP provisions exactly that.  

When your application needs more resources, CP scales it up. When it needs less, CP scales it down.  
So you're not overprovisioning just because you can't manage resources granularly.  
And you're not paying for idle capacity.  

You provision exactly what you need and the CP platform adjusts it automatically based on actual needs.  
CP claims that customers typically see 60 to 80% reduction in cloud compute costs compared to running and 
overprovisioning their own K8s and surrounding infrastructure directly on the cloud providers platforms.  

The savings come not because the underlying cloud services are cheaper, but because you're not wasting capacity 
and paying for something you're not using, plus you're not paying for all the engineering overhead of stitching 
everything together manually.  

# Hands-On Demo: Multi-Cloud Deployment

## Demo Overview

In this demo, we're going to:
- take an example application, which is a very simple JavaScript application with a NodeJS backend
- dockerize it (building a Docker image out of it)
- push the Docker image to the DockerHub registry
- deploy that image on multiple cloud platforms at once (Azure, AWS and GCP) in Kubernetes clusters

There are two interesting details about that application:
- it connects to an S3 bucket (on Nana Janashia's AWS account) to fetch some data, and displays it in the UI
- it also displays from which region of which cloud platform it's serving the data request

## Application Overview

- repo: https://gitlab.com/twn-youtube/multi-cloud-crash-course

At the root of Nana's project, we have an app.js file that contains the backend code.  
The only thing this application does is connecting to an S3 bucket (the bucket name is at line 16 in the app.js file).  
The app fetches the contents from this bucket and sends them to the frontend.  

The frontend part is located inside the public folder of Nana's repo.  
For each file stored in the bucket, the frontend displays their metadata.  

The only dependencies for this app are specified in the package.json file.  

## Step 1: Setting up the Multi-Cloud Infrastructure

We need to provision the environment on which we want to deploy our dockerized application.   

In Control Plane UI, we're going to create a "GVC" = Global Virtual Cloud, which behaves like one unified cloud.  
A GVC defines a single logical cloud environment that spans across multiple regions and/or multiple cloud providers.  

When creating a GVC, we name it, add a description, and then we go to "Locations" to select different locations on different cloud providers.  
There, we're going to select 3 different cloud providers and the locations that suit our needs.  

When we click on Create to create our GVC, Control Plane will provision selected locations for us, without any setup effort.  
So we can get started without having to connect our own cloud providers.  

And just like that, we have a global cloud that is made up of multiple cloud providers in different regions.  
Of course, there's a lot of technical complexity behind that apparent simplicity.  

Our infrastructure is provisioned by CP in the background, including all necessary setups:
- it creates VPCs and VNets (Virtual Private Clouds and Virtual Networks)
- it creates subnets
- it configures routing tables
- it creates security groups/firewall rules
- it provisons Kubernetes clusters
- it deploys load balancers
- it configures autoscaling groups

Control Plane orchestrates an unlimited number of hardened, security-isolated Kubernetes clusters in all regions and for all major clouds.  

Our infrastructure and Kubernetes clusters are provisioned on these underlying cloud providers with all the settings required, so that 
we can literally throw our Docker containers into it and it's going to run them in Kubernetes clusters.  

## Step 2: Deploying the Application

Now, let's dockerize our application.  
For that, we need to build a Docker image from which we'll be able to spin up containers that will run our application.  

The Dockerfile that will build the Docker image for our app can be found in Nana's repo: https://gitlab.com/twn-youtube/multi-cloud-crash-course  

It's a very simple Dockerfile that does the following:
- copies the package.json file to an Alpine Linux instance that has NodeJS installed
- uses that file to install production dependencies
- copies the NodeJS backend code
- copies the frontend code ("public" folder)
- exposes port 8080 (which is the port on which our app is listening, as defined in the backend code) 
- starts the application, which translates to running `node app.js`

Let's build that image and push it to DockerHub:  
22/39

# Final thoughts

## What happens if Control Plane goes down ?

Your workloads continue running — Control Plane going down doesn't take your app down with it.  
Think of it like Kubernetes itself: if the API server goes down, your pods don't stop.  
The infrastructure keeps serving traffic. You just temporarily lose the ability to make new changes until it recovers. 

