In the [Part 1](https://techmusings.dev/buildingACloudMigrationPlatformPart1ProvisioningTheInfrastructure) of the blog, we saw the motivation behind the project and 
how we built the Step Executor which would be responsible for execution of Terrform scripts.  We also saw how the Step Executor would take the following inputs

* Trace Id
* Auth Information
* Step To Execute
* Inputs ( Key Value ) to be passed while executing the Terraform

In this post we'll look into the **Orchestrator** which helps in orchestration of multiple terraform scripts and provide the collective results back to the provisioning request. 

## TLDR
A component creation may need other pre-requisites to be created. The orchestrator, identifies the execution plan to create the component by orchestrating it's dependencies before creating the intended component. The design is motivated from how a CI/CD pipeline is built with multiple steps and running them in series.

## Component dependencies

A typical use case we were trying to solve was a request for components such as a Cloud SQL DB. The creation of this component will require some user inputs such as 

* Type of Database
* Database Name
* Admin user name
* Project under which it needs to be created
* DB Version
* Charset
* Collation
* Disktype
* Activation policy 
etc... 

In these inputs, some of these an be safely defaulted at the organization level, but keeping thouse defaults as part of terraform script would be a bad idea as it would bring in a tight coupling. Similarly some of the inputs cannot be defaulted and needs to flow in as user inputs. 

Apart from this, the creation of the component is hardly standalone in nature. For example, the postgres DB would require a VPC and Subnet to be created before provisioning the DB itself. Which means that either the terraform script of the DB should also have provision for creation of these prerequisites, so that they all run in sequence together. But in most cases, the VPC and subnets are shared across multiple DBs and server instances to have network isolation and accessiblity. So keeping these component provisioning as part of the same terraform script would create a tight coupling. 

Therefore there were 2 important considerations while we wanted to create a component

1. Safe defaults at the organization level for each component
2. List of dependent infra components to be provisioned before creating the intentended component.

To solve this, we designed and calculated something called the "Execution Plan" before every component's provisioning. The main purpose of the Orchestrator was to capture, calculate and orchestrate the execution plan. 


## Execution Plan





## Input Dependencies



## Collating outputs




## Changes to Step Executor



## Conclusion
