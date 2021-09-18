In the [Part 1](https://techmusings.dev/buildingACloudMigrationPlatformPart1ProvisioningTheInfrastructure) of the blog, we saw the motivation behind the project and 
how we built the Step Executor which would be responsible for execution of Terrform scripts.  We also saw how the Step Executor would take the following inputs

* Trace Id
* Auth Information
* Step To Execute
* Inputs ( Key Value ) to be passed while executing the Terraform

In this post we'll look into the **Orchestrator** which orchestrates the creation of the components. 
