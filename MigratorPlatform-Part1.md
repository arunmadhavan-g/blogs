# Building A Cloud Migration Platform - Part 1 : Provisioning the infrastructure


A recent change in our Org, all the applications are required to be moved to GCP as part of a strategic decision. 
Most of our apps run out of Azure today with a few services deployed in AWS as well, with tons of other apps running out of on-prem datacenters. 


There has been lots of work done by different people across different teams to migrate their apps into GCP. Though there is generally good sharing of information, 
the loss of information exchange is also quite common due to the shear size of the organization. Another major problem is lack of working knowledge in GCP, which is an org wide issue. 
Not every one is aware of the best practices that needs to be followed as part of their provisioning of infrastructure, which components to choose etc. and there still exists a lot of learning curve to go through. 

Recently as part of the GCP adoption initiative, we had a hackathon conducted and we pitted in with a solution to make things simple for teams to move to GCP. 
I'll share my thoughts on how we approached the problem statement as part of this blog series. As part of this blog, we will see how we approached the problem of provisioning the infrastructure components in GCP. 


## TLDR;

We built a stateless microservice - "Step Executor", that takes in request to trigger a Terraform script and respond back with the step output. 
A GCP component provisioning might need provisioning of other components as pre-requisites. These we modelled it as a Plan with multiple steps. 
Each step is a terraform script, and a plan consists of multiple smaller steps. 
The execution of a plan and it's many smaller steps is handled by another microservice called the "Orchestrator" that executes the steps in sequence.


## Overall Application

We planned to build a platform that would be able to 

* Understand the application that needs migration
* Identify the current components used and the infrastructure in which they are deployed ( Azure/ AWS/ On-Prem )
* Suggest GCP specific alternatives, and come up with a migration plan
* Auto provision the components identifed
* Provide a platform that can also act as a source for all the application information ( like a keyvaule ) 

So the flow can be broadly classified as 


Identify  -> Suggest -> Provision -> Collate and Serve





