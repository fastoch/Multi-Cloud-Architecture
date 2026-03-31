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

## 1. 

---
4/39
