In the [Part 1](https://techmusings.dev/buildingACloudMigrationPlatformPart1ProvisioningTheInfrastructure) of the blog, we saw the motivation behind the project and 
how we built the Step Executor which would be responsible for execution of Terrform scripts.  We also saw how the Step Executor would take the following inputs

* Trace Id
* Auth Information
* Step To Execute
* Inputs ( Key Value ) to be passed while executing the Terraform

In this post we'll look into the **Orchestrator** which helps in orchestration of multiple terraform scripts and provide the collective results back to the provisioning request. 

## TLDR
A component creation may need other pre-requisites to be created. The orchestrator, identifies the execution plan to create the component by orchestrating it's dependencies before creating the intended component. The design is motivated from how a CI/CD pipeline is built with multiple steps and running them in series.

## Components have dependencies


## Execution Plan



## Input Dependencies



## Collating outputs




## Changes to Step Executor



## Conclusion
