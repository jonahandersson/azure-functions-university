# Azure Durable Functions - Introduction & Function Chaining Pattern (.NETCore C#)

Watch the recording of this lesson [on YouTube]() //TODO add appropriate link

## Goal 🎯

The goal of this lesson is to give you an introduction into Azure Durable Functions.
In this lesson you will learn how to create your a serverless function app using Azure Durable Functions using chaining pattern.

This lessons consists of the following exercises:

|Nr|Exercise
|-|-
|0|[Prerequisites](#0-prerequisites)
|1| Introduction to Serverless and Azure Durable Functions 
|2| How to Create a Durable Function App project in .NET Core using VS Code 
|3| How to Create a Durable Function App project in .NET Core using Visual Studio 
|4| How to Create a Durable Function App project in .NET Core using Azure Portal
|5| Function Chaining Example with Azure BLOB Trigger
|6|	Implementing a "Real-World" Scenario
|7|	Retries - Dealing with Temporal Errors
|8| Circuit Breaker - Dealing with Timeouts
|9|[Homework](#4-homework)
|10|[More info](#5-more-info)

> 📝 **Tip** - If you're stuck at any point you can have a look at the [source code](../../src/{language}/{topic}) in this repository.

---

## 0. Prerequisites

| Prerequisite | Exercise
|-|-
| Install latest [.NET Core SDK](https://dotnet.microsoft.com/download) if you don't have it. | 2,3,5,6,7,8
| Install [VS Code](https://code.visualstudio.com/download) if you don't have it. | 2,5,6,7,8
| Install [Azure Functions extension for VSCode](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions). | 2,5,6,7,8
| Install [C# extension for VSCode](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp). | 2,3,5,6,7,8
| Install latest version of [Azure Core Tools](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cportal%2Cbash%2Ckeda) | 2,3,5,6,7,8
| Latest [Visual Studio](https://visualstudio.microsoft.com/vs/community/) | 2
| [Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/) | 2,5,6,7,8
| A local folder for the Function App. | 2-5


See [{language} prerequisites](../prerequisites/prerequisites-{language}.md) for more details.

> 📝 **Tip** - Up to now the Durable Functions are not compatible with Azurite with respect to the emulation of storage. So if you are on a non-Windows machine you must use a hybrid approach and connect your Durable Functions to a storage in Azure. This means that you need an Azure subscription.

## 1. Introduction to Azure Durable Functions

Within this section we want to take a look at the motivation for the usage of Azure Durable Functions and take a look at the underlying mechanics.

### 1.1 Functions and Chaining

In general, Functions are a great way to develop functionality in a serverless manner. However, this development should follow some guidelines to avoid drawbacks or even errors when using them. The three main points to consider are:

* Functions must be stateless
* Functions should not call other functions
* Functions should only do one thing well

While the first guideline is due to the nature of functions, the other two guidelines could easily be ignored but would contradict the paradigms of serverless and loosely coupled systems. In real life scenarios we often have to model processes that resemble a workflow, so we want to implement a sequence of single steps. How can we do that sticking to the guidelines? One common solution for that is depicted below:

Every function in the picture represents a single step of a workflow. In order to glue the functions together we use storage functionality, such as queues or databases. So Function 1 is executed and stores its results in a table. Function 2 is triggered by an entry in the table via the corresponding bindings and gets executed representing the second step in the workflow. This sequence is then repeated for Function 3. The good news is, that this pattern adheres to the guidelines. But this pattern comes with several downsides namely:

* The single functions are only coupled via the event that they react to. From the outside it is not clear how the functions relate to each other although they represent a sequence of steps in a workflow.
* The storage functionality between function executions are a necessary evil. One motivation for developing serverless is to care about servers less. Here we must care about the technical infrastructure in order to have our functions loosely coupled.
* If you want to pass a context between the functions you must store it (and deal with the potential errors around it).
* Handling errors and analyzing bugs in such a setup is very complicated.

Can we do better? Or is there even a solution provided by Azure Functions to handle such scenarios? There is good news - there are Azure Durable Functions.

### 1.2 Solution via Durable Functions

Azure Durable Functions is an extension to the Azure Functions Framework that allows you to write workflows as part of your Azure Functions code. </br>
Although queues and table storage are still used, the Durable Functions extension abstracts those away, so that you can focus on the business requirement at hand. 
The function state is managed by making use of the [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) pattern. In addition the extension helps you with common functionalities in workflows, such as retries and race conditions, as we will see later. Let us first take a look at how Durable Functions work and introduce some terminology.

### 1.3 Mechanics of Durable Functions

Durable Functions uses three types of functions:

* Orchestrator Functions: the central part of the Durable framework that orchestrates the actions that should take place by triggering Activity Functions.
* Activity Functions: the basic workers that execute the single tasks scheduled via the Orchestrator Function.
* Client Function: the gateway to the Orchestrator Function. The Client Function triggers the Orchestrator Function and serves as the single point of entry for requests from the caller like getting the status of the processing, terminating the processing etc.

Let us assume the following simple execution sequence saying Hello to different cities with three tasks triggered by an HTTP request:

```csharp
[FunctionName("E1_HelloSequence")]
public static async Task<List<string>> Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var outputs = new List<string>();

    outputs.Add(await context.CallActivityAsync<string>("E1_SayHello", "Tokyo"));
    outputs.Add(await context.CallActivityAsync<string>("E1_SayHello", "Seattle"));
    outputs.Add(await context.CallActivityAsync<string>("E1_SayHello_DirectInput", "London"));

    // returns ["Hello Tokyo!", "Hello Seattle!", "Hello London!"]
    return outputs;
}
```
The second task depends on the result of the first task.

The schematic setup with Azure Durable Functions looks like this:

The Client Function is triggered by an HTTP request and consequently triggers the Orchestrator Function. Internally this means that a message is enqueued to a control queue in a task hub. We do not have to care about that as we will see later.

<img>Illustration image here

After that the Client Function completes and the Orchestrator Function takes over and schedules the Activity Function. Internally, Durable Functions fetches the task from the control queue in the task hub to start the Orchestrator and enqueues a task to the work-item queue to schedule the Activity Function.

<img>Illustration image here

The execution of the Orchestrator Function stops once an Activity Function is scheduled. It will resume, and replay the entire orchestration once the Activity Function is complete.

<img>Illustration image here

When the Orchestrator Function is replayed it will check if there are tasks (Activity Functions) left to execute. In our scenario the second Activity Functions is scheduled. This cycle continues until all Activity Function calls in the Orchestrator have been executed.

After this theoretical overview it is time to make our hands dirty and write some code!

## 2. Creating a Function App project for a Durable Function

Our scenario comprises a Durable Function App with one Activity Function. The app will be triggered via an HTTP call. The Activity Function receives a city name as input and returns a string in the form of "Hello _City Name_" as an output. The Activity Function is called three times in sequence with three different city names. The app should return the three strings as an array.

### 2.1 The Client Function

The first function that we create is the Client Function of our Durable Function app that represents the gateway towards the Orchestrator Function.

#### Steps

1. Create a directory for our function app and navigate into the directory.

   ```
   ```

2. Start Visual Studio Code.

   ```
   ```

3. Create a new project via the Azure Functions Extension.
   1. Name the project `DurableFunctionApp`.
   2. Choose `CSharp` as language.
   3. Select `Durable Functions HTTP Starter` as a template as we want to trigger the execution of the Durable Function via an HTTP call.
   4. Name the function `DurableFunctionStarter`.
   5. Set the authorization level of the function to `Anonymous`.

### 2.2 The Orchestrator Function

Now we create the Orchestrator Function, which is responsible for the orchestration of Activity Functions.

#### Steps

1. Create a new function via the Azure Functions Extension in VSCode.
   1. Select `Durable Functions orchestrator` as a template.
   2. Name the function `DurableFunctionsOrchestrator`.

   > 📝 **Tip** - Remove the comments from the index.ts file. The code of the Orchestrator Function should look like this:

   ````csharp
 
   ````

   > 🔎 **Observation** - Take a look into the `function.json` file of the Orchestrator Function. You find the binding type `orchestrationTrigger` which classifies the function as an Orchestrator Function.

   > ❔ **Question** - Can you derive how the Orchestrator Function triggers the Activity Functions? Do you see potential issues in the way the functions are called?

### 2.3 The Activity Function

To complete the Durable Function setup we create an Activity Function.
#### Steps

1. Create a new function via the Azure Functions Extension in VSCode.
   1. Select `Durable Functions activity` as a template.
   2. Name the function `HelloCity`.

   > 📝 **Tip** 

   ````csharp
 
   ````


   > 🔎 **Observation** - 

   > ❔ **Question** - If you trigger the orchestration now, would you run into an error? What adoption do you have to make (hint: function name in orchestrator)?

### 2.4 The First Execution

Execute the Durable Function and experience its mechanics.

#### Steps

1. Start the Azure Storage Emulator.
2. Set the value of the `AzureWebJobsStorage` in your `local.settings.json` to `UseDevelopmentStorage=true`. This instructs the Azure Functions local runtime to use your local Azure Storage Emulator.
3. Start the function via 
   > 🔎 **Observation** - You can see that the runtime is serving three functions.

   
4. Start the Client Function via the tool of your choice e.g. Postman.
   > ❔ **Question** - What route do you have to use?

   > 🔎 **Observation** - The result of the orchestration is not directly returned to the caller. Instead the Client Function is returning the ID of the orchestrator instance and several HTTP endpoints to interact with this instance.
     

5. Call the `statusQueryGetUri` endpoint to receive the results of the orchestration.

     > 🔎 **Observation** - The status endpoint returns not only the result but also some metadata with respect to the overall execution.

6. Check the resulting entries in your Azure Storage Emulator
   > ❔ **Question** - How many tables have been created by the Azure Functions runtime? What do they contain?

## 3. Implementing a "Real-World" Scenario

In this section we develop some more realistic setup for our Durable Function. Assume the following situation:
Assume that we have the name of a GitHub repository. Now we want to find out who is the owner of this repo. In addition we want to find our some more things about the owner like the real name, the bio etc. To achieve this we must execute two calls in a sequence to the GitHub API:

1. Get the information about the repository itself containing the user ID.
2. Based on the user ID we fetch the additional information of the user.

The good thing about the GitHub API is that we do not need to care about authentication and API keys. This means that there are some restrictions with respect to the allowed number of calls per minute, but that is fine for our scenario.
### 3.1 Basic Setup of "Real-World" Scenario

In this section we add the skeleton for the implementation. This time we do not need to create a Client Function again, because we will reuse the one of our prior example. This is possible as the Client Function forwards the requests based on the Orchestrator Function name in the URL of the HTTP call.

#### Steps

1. Create a new function via the Azure Functions Extension in VSCode.

2. Create a new function via the Azure Functions Extension in VSCode.

3. Create a new function via the Azure Functions Extension in VSCode.
 
4. Install the following npm modules for a smooth interaction with the GitHub REST API

### 3.2 Implementation of the Orchestrator Function

In this section we implement the Orchestrator Function that defines the call sequence of the Activity Functions and assures the transfer of the result of the first Activity Function to the second one.
#### Steps


1.
2. 
3.

### 3.3 Implementation of the Activity Function `GetRepositoryDetailsByName`

In this section we implement the Activity Function `GetRepositoryDetailsByName`. We fetch the data of the repository by name making use of the `/search/repositories` endpoint.


## 6. Homework

[Here]() is the assignment for this lesson.

In addition we also have an additional homework that deals with a more advanced scenario i. e. making use of the SAP Cloud SDK to call a downstream SAP system. You find the instructions [here]().

## 7. More info

* Azure Durable Functions - [Official Documentation](https://docs.microsoft.com/en-us/azure/azure-functions/durable/)
* C# .NET Documentation - https://docs.microsoft.com/en-us/dotnet/csharp/
* Azure Durable Functions - [Automatic retries](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-error-handling?tabs=javascript#automatic-retry-on-failure)
* Azure Durable Functions - [Function timeouts](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-error-handling?tabs=javascript#function-timeouts)
* More info on the [circuit breaker pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
* [GitHub REST API](https://docs.github.com/en/free-pro-team@latest/rest)
* Alternative to code-based workflows in Microsoft Azure: [Azure Logic Apps](https://azure.microsoft.com/en-us/services/logic-apps/)

---
[🔼 Lessons Index](../../../README.md)

