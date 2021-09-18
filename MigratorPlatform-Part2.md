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

I had mentioned that the overall design was inspited by a CI/CD pipeline where a pipeline can have many tasks which configured to be exeuted in parallel or series and often there are cases where the output from a previous step are used as input in subsequent steps. 


Similarly we designed our system to have a root component called the "Execution Plan" or just a Plan for brievity. The execution plan consists of individual steps. Each step in this case are individual terraform executions ( executed by Step Executor ). A Plan will have user inputs, which as discussed earlier will have definition of all possible inputs to the creation of the component.  


Similarly each step will have step inputs, which would be passed down to the Step executor for executing the terraform script. Interestingly the step's input can come from either the plan's input or from the output of each step. Thus the output from each of these steps were also defined. 
A step input mapping was defined to tell from where the information would come from for the step. 


Finally the complete plan's output was captured which are basically outputs from individual steps. Our overall model looked like below. 

![](./images/migrationPlatform/ExecutionPlan-ER.svg)


## Handling the inputs

By defining the Execution Plan, we were able to solve the issue with component pre-requisite dependencies. We still had to provide the system the ability to provide defaults. For this, we captured for every plan's input a Plan Input Default values. 

We also realised that some of the inputs being available or absent would change our execution plan. For example, A VPC or a subnet value could be passed as a user input dependening on whether or not it's already available. Their presence or absence would lead to changes to the execution plan where we may or may not need some of the dependent steps to be available. 


So we decided to have a nullable field with reference to the step id in the plan input. This helped us identify the step that can be avoided in case the value gets supplied. An handling was done at the Step Input level to pass value from "user input" rather than from the "previous step output" in these cases based on this information as well. 


These changes helped us in 

* Fully flexible Terraform scripts with no dependencies in the script
* The defaults to values were stored as part of the table, keeping it configurable
* Calculate the Execution Plan appropriately depending on specific input value availability. 


## Executing the Plan

With the above changes, we were able to successfully capture the information requires to identify a plan and execute it. Now while executing the plan, it is done at a request level. Every request would have to be captured uniquely, which includes, the Plan Inputs, Step Inputs that were passed as part of each step's execution, step outputs and the plan output. 


For this reason, we created what we call the "Value" tables, which would be used to capture the above mentioned information. All the information across these tables were linked together using a "traceId" which is passed from the caller's location ( Mediator ). The traceId is also passed down to the Step Executor, so that when it calls back the "Orchestrator" it would be able to tell for which step belonging to which traceId was executed. 

Once all the steps in the Plan were executed, the Plan output values were populated and a call is made back to Mediator with the traceID and the output values of the Plan. 


## Changes to Step Executor

We hacked together a bunch of code to make all this work. Created a spring boot application and created the tables with Flyway migration, persisting our data into a postgres DB. While doing so, we continued to keep the stepName consistent with Step Executor and Orchestrator so that the Step Executor would be able to store all the Terraform scripts inside itself, while we ensure passing of the correct name for step executor the pull the correct script to execute it. 

But as we started splitting, refactoring and adding more scripts, we realised that human error is something that cannot be avoided and this is not going to scale. 


To handle this, we decided to move the scripts to a google object store. The location of the script was captured as part of the step and was passed down to the "Step Executor" as input. The step executor's body changed as shown below. 


We made the step executor to download the script from the object store everytime it executes, there by eliminating storing the scripts as part of the code base. The step executor is now a stateless component which worked based on the inputs that was passed to it.  The below sequence explains on how the overall execution happened. 





## Continuation

Orchestrator was able to capture the plan's information also handle the orchestration of individual component creation request. We did not develop any specific UI for the same and handled the values using migration scripts. In our next part, I'll talk about how we managed to provide user interaction using the mediator service. 



