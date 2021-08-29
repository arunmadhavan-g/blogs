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

So the flow can be broadly represented as 

![Overall Flow](./images/migrationPlatform/OverallFlow.svg)


Our overall architecture had the below components 

![Overall Architecture](./images/migrationPlatform/overall-architecture.svg)

| Service/ Component | Description | 
|---|---|
| UI | A react based UI application, that was used for user interactions |
| Mediator | A spring boot application, which acts as the backend for the UI, and is responsible for identifying the components from the user and works with other services to identify and provision the components. |
| Suggestion Engine | An engine that takes the current application information to suggest a migration plan. Built as a spring boot application this integrated with many other services for analysis and prediction | 
| Orchestrator | A spring boot service, that takes a migration plan, and orchestrates the creation of components in GCP |
| Step Executor | The Node and Typescript application that triggers terraform scripts for the actual provisioning of components, based on the inputs passed from Orchestrator | 


In this blog post, we'll specifically how we designed and developed the Step Executor. 


## Provisioning of GCP Components

We decided that to automate the provisioning process, it is essential that we codify the process. We had options to choose from and the major 2 options that we wanted to leverage were **Terraform** and **Google Cloud Deployment Manager**. We decided to go with terraform as it would provide a way to make the solution platform agnostic. So in case of a shift in company strategy in future to choose a different cloud service, the platform would require re-work only on the Terraform scripts, with the rest of the services being intact. 

Typically a component provsioning might also need provisioning of related components, before the actual component is provisioned. For example, if one of the requirement is to create a Postgres DB in GCP's Cloud SQL, it would require creation of VPC and Subnet inside which the DB needs to be provisioned to provide it with required level of network isolation and security.  So this would mean that for spinning up a DB I need to 

1. Provision a VPC 
2. Provision a Subnet inside the VPN
3. Pass the subnet information while Spinning up the DB

We call this an execution plan or to put it simply **Plan**. A plan consists of many individual steps. In the case above, the plan has 3 steps, viz. Provisiong of  VPC, Subnet and DB. A step can have inputs that are passed from the user, or from one of it's previous steps. 

For Example, a database provisioning needs the Size and Name of the DB, that would come from the user, whereas the subnet under which it needs to get created would come from it's previous step. 


Addition to the inputs from the user provisioning of infrastructure also needs security. Each team needs to isloate their components from the others and they will behave like an org of their own. So each team will have to pass their own security for provisioning their components. 

So in short, during each step's execution, terraform would take the user/step inputs and security details to execute the script and provision a single component.

With the above information in hand, let's take a look into how we were able to pull off an actual implementation. 


## Step Executor

Step Executor was a service that exposed a single POST endpoint, to which we can pass a set of input and security information, based on which it would choose and run a terraform script. The endpoint would handle the request in an async fashion and will make a call back to a webhook endpoint exposed at the "Orchestrator Service" ( to be discussed later ) with the output from terraform. 


### Terraform Rules
To ensure uniformity across all the different scripts we wanted to set the following rules for terraform scripts

1. Every script is used to create a single component
2. Every script takes a set of input and exposes a set of output 
3. All the terraform scripts would read credentials from a credentials file called `auth.json`
4. The inputs are directly passed to the `terraform plan` or `terraform apply` commands
5. The output of the `terraform apply` command is written into `terraform.tfstate` state file 

Below is a sample terraform file that we used for creating a postgres DB in Cloud SQL along with an admin user

```
provider "google" {
    credentials = file(var.credentials_file)
    project = var.project
}

provider "google-beta" {
    credentials = file(var.credentials_file)
    project = var.project
}

variable "project" { 
	default = ""
}

variable "credentials_file" {
	default = "auth.json"
}

variable "region" {
  default = "us-central1"
}

variable "zone" {
  default = "us-central1-c"
}

# database instance settings
variable db_version {
  default = "POSTGRES_11"
}

variable db_tier {
  default = "db-f1-micro"
}

variable db_activation_policy {
  default = "ALWAYS"
}

variable db_disk_autoresize {
  default = true
}

variable db_disk_size {
  default = 10
}

variable db_disk_type {
  default = "PD_SSD"
}

variable db_pricing_plan {
  default = "PER_USE"
}

variable db_instance_access_cidr {
  default = "0.0.0.0/0"
}

# database settings
variable db_name {
  description = "Name of the default database to create"
  default = "default_db"
}

variable db_charset {
  description = "The charset for the default database"
  default = ""
}

variable db_collation {
  description = "The collation for the default database. Example for MySQL databases: 'utf8_general_ci'"
  default = ""
}

# user settings
variable db_user_name {
  description = "The name of the default user"
  default = "admin"
}

variable db_user_host {
  description = "The host for the default user"
  default = "%"
}

variable db_user_password {
  default = ""
}

output "connection_name" {
  value       = google_sql_database_instance.postgresql.*.connection_name
  description = "Postgress Connection Name"
}

resource "google_sql_database_instance" "postgresql" {
  
  provider = google

  name = "postgresql"
  project = var.project
  region = var.region
  database_version = "${var.db_version}"
  
  settings {
    tier = "${var.db_tier}"
    activation_policy = "${var.db_activation_policy}"
    disk_autoresize = "${var.db_disk_autoresize}"
    disk_size = "${var.db_disk_size}"
    disk_type = "${var.db_disk_type}"
    pricing_plan = "${var.db_pricing_plan}"
    
    location_preference {
      zone = var.zone
    }
   
    ip_configuration {
      ipv4_enabled = "true"
      authorized_networks {
        value = "${var.db_instance_access_cidr}"
      }
    }
  }
}

# create database
resource "google_sql_database" "postgresql_db" {
  provider = google-beta

  name = "${var.db_name}"
  project = "${var.project}"
  instance = "${google_sql_database_instance.postgresql.name}"
  charset = "${var.db_charset}"
  collation = "${var.db_collation}"
}

# create user
resource "random_id" "user_password" {
  byte_length = 8
}

resource "google_sql_user" "postgresql_user" {

  provider = google-beta

  name = "${var.db_user_name}"
  project  = "${var.project}"
  instance = "${google_sql_database_instance.postgresql.name}"
  host = "${var.db_user_host}"
  password = "${var.db_user_password == "" ?
  random_id.user_password.hex : var.db_user_password}"
}
```
The final execution consisted of 

* Creating a `temp` Folder
* Moving the terraform file tat needs to be executed
* Creating an `auth.json` based on the security information passed
* Executing `terraform init` to iniaitlize all the plugins required for executing the script
* Executing `terraform plan -vars...` for creating the execution plan
* Executing `terraform apply -vars...` to actually provision the resources


### Implementation 

With the above steps defined, we did an implementation using NodeJs and Typescript, exposing a POST endpoint. It took the following input as request body

```
{
	"traceId": "traceID from the requesting system", 
	"stepName": "name of the terraform that needs to be executed", 
	"auth": { "type": "service_account",
	  "project_id": "<project id from GCP>",
	  "private_key_id": "<keyId>",
	  "private_key": "<Private Key>",
	  "client_email": "<service email>",
	  "client_id": "<GCP Client Id>",
	  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
	  "token_uri": "https://oauth2.googleapis.com/token",
	  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
	  "client_x509_cert_url": "<client-cert-url>"
	},
	"inputs": [{
		"label":"input key as per terraform script",
		"value": "value for the given label"
	}, ...]
}
```

Each of the inputs defined in the terraform script that we desire to execute are captures as a key-value pair defined by "label" and "value" in the `inputs` and are passed as part of the request. 



To start with, we created a `script` folder inside the code base, which had all the terraform scripts, and the input stepName would have to match with the name of the terraform script that needs execution. As mentioned earlier, we implemented code using `childProcess`'s `exec` command to execute the various steps of terraform. At the end of every execution the outputs from `terraform.tfstate` file was read after which the temp folder would be deleted. 

On successful completion of terraform and on reading the output a PUT call is made to the "Orchestrator service's" endpoint `<baseURL>/planstepstatuses/{traceId}` with the "label-value" format of output.

### Packaging the code

Terraform was executed as a command line script using `exec` as mentioned above. In any server it needs to run, it should have terraform pre-installed. We packaged the code as a docker image, with terraform as it's base image. This ensured that terraform is always installed and ready to be executed inside the docker container. 

We installed node into the docker container and moved the code inside the same exposing the port 8000 for receiving calls from external systems. 
Our docker file looked something like below. 

```
FROM hashicorp/terraform:light

RUN apk add --update nodejs npm

COPY ./scripts ./scripts
COPY ./src ./src
COPY package.json .
COPY package-lock.json .
COPY tsconfig.json .

RUN npm install 

EXPOSE 8000
ENTRYPOINT [ "npm", "run", "start" ]
```

This helped us execute the scripts in a self sustained manner. We pushed this docker imaged into [Google Container Registry](https://cloud.google.com/container-registry)

### Deploying and Running the Application

Our initial thought was to make use of the [Google Kubernetes Engine(GKE)](https://cloud.google.com/kubernetes-engine) to run the docker image. Please note that this was part of the GCP hackathon and we were choosing components from Google's Cloud offering.  

But we later realised that running a dedicated service would not make sense as the number of times the provisioning will be triggered would be relatively very low. Having a dedicated service for the same would have been costly. So we wanted to take a serveless approach, where we would be able to execute the Step Executor when ever needed. 

The Step Executor is also expected to operate in an async fashion. As mentioned earlier, on completion of the step, it makes a call to the endpoint exposed by Orchestrator, which makes is a perfect candidate for async execution. 

So we explored a way to execute the docker in a serverless fashion. That is when we came across [Google Run](https://cloud.google.com/run) which 

* Enables running of Docker images as a serverless proces
* Point to the GCR repository for taking care of automatic deployment
* Enables triggering of the process through [Cloud PubSub](https://cloud.google.com/pubsub)

This looked ideal.  The trigger from PubSub required some minor modifications to the code, as the message posted is sent as a Base64 encoded content to the POST endpoint.  Our Deployment architecture of Step Executor looked like below. 


![step executor arch](./images/migrationPlatform/step-executor-deployment-arch.svg)


## Continuation

We were able to post sample json request to the PubSub topic that we had created and was able to provision the resources successfully. 
I'll continue how we implemented the "Orchestrator" as next part of my blog where the step executor also went through a few changes. 











