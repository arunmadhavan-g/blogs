We had to, for one of our clients  help them integrate Business process management(BPM) into their existing multi-tenant system. 
The solution had to 

* Easily import a standard BPMN 2.0 based definition and also let create one as well.
* Work with multiple tenants and ensure data does not get shared between user of tenants.
* Open source to keep the cost involved low. 
* Easy to scale.
* Customizable to meet their existing platform requirements. 

The rest of the talks about how we managed to accomplish that and the thought process that went into the building of the overall solution. 

# TL;DR
   
We integrated an Open Source Java based BPM system [Camunda](https://camunda.com/), that provides a way to add the complete platform as a 
spring application. Also we did not store any user information into the Camunda system and leveraged it's APIs to

1. Maintain the BPMN definition at the source system.
2. For every need for a business process a new Deploy of BPMN definition is done at Camunda
2. A task is then created to trigger the business process for the deployment. 
3. The deployment and task IDs were maintained in the source system
4. Make call backs using Listeners to notify when the task is completes to the source system.
5. Un-Deploy when the process completes. 

This helped us to keep things alive for a specific approval process and also helped maintain complete isolation between processes. 
This also potentially cleared all the headaches of maintaining the users across two different systems. 

# BPMN : A quick intro

Let us consider an editorial example that involves the following work flow. 

* An employee writes a piece of article and submits for review
* The article goes through a review engine, that based on the key words flags for sensitive material
* Based on the flagging level the article would go to  
    * Sub Editor in case of a normal one
    * Senior editor in case it's sensitive
* The reviewer looked into the article and can 
    * Approve it if it's good
    * Send back if there are changes needed
    * Reject it if it's not worth publishing
* After the above process the article gets
    * published and available for others to read if Approved
    * sent back to the employee who can rework on it and go through the process again
    * moves to the archive in case of Rejected

Typically we have a flow that can be defined here. A similar parallel can be drawn against many such situations which also works with enterprises and processes such as 

* Leave approval flow ( where it can be an immediate manager who would approve it )
* Procurement ( where there can be many levels of approval )
* Travel Requests  ... 


Typically the processes are defined by the business and there will be a team who would work on translating it to code. 
A BPM system acts as a bridge to make things more accessible to both the parties. 

It allows the business or the business analyst to draw the BLOCKS of and the connections between them. This is defined in the background as a XML following a standard called the BPMN2.0 notation.

// TODO: Insert the BPMN simple thing image here. 


# Why Camunda


# Camunda as Spring Application


# User Management Conundrum


# Keeping things stateless


# Final Solution


# Summary
