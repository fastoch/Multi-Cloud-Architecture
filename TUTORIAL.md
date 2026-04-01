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




---
8/39
