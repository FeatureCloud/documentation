# FC App Development Documentation

## Introduction

FeatureCloud's primary goal is to simplify the development and usage of federated machine learning algorithms. This involves three major challenges: development of apps, distribution of apps, and usage of apps.

FeatureCloud uses Docker as a virtualization technique, to ensure system access of federated apps is as limited as possible. This way FeatureCloud minimizes risks such as leakage of sensitive data or access of data, which should not be included in a machine learning algorithm (e.g. due to lack of consent). 

In order to execute on the FeatureCloud platform, a federated app must implement the FeatureCloud API. The API is designed in a generic way, it puts minimal constraints on the actual implementation of the app, so any kind algorithm can be implemented.

It is important that the app is able to act both as coordinator or participant. This is needed because the same app is downloaded by the platform to all sites, and the app's role (coordinator or participant) is determined at setup. Therefore, a coordinator is a special participant, that receives the local results of all participants and broadcasts the aggregated results back.

## The FeatureCloud API

A federated app should act as a web server which is polled by the FeatureCloud platform and responds to the FeatureCloud API. It should provide the following requests, that are already integrated in the FeatureCoud app templates.

### POST /setup
When the participants are ready to start the federated executio the platform will send the setup request.
This is the starting point of the federated execution, the app can use it as a trigger to start the computation based on it's local data.
In this call, each participant gets an id, the number of all participants and the information if they are the coordinator (master).
The request body contains the following information:
- `id (string)`: the app instance identifier, determined by the platform
- `master (boolean)`: this value specifies the role of the app instance: true for coordinator, false for participant
- `clients (array of string)`: contains the identifiers of all participants
 
Example of setup data for a coordinator, when there are 3 participants in total:
`{
  id: "0",
  master: true,
  clients: ["0", "1", "2"]
}`
 
### GET /status

With the response to this request the federated app reports its current status. It is frequently sent and  indicates if there is data to be transferred to the coordinator (from participant), data to be broadcasted to the participants (from coordinator) or if the execution of the app is finished.
The	response should contain the following information:
- `available (boolean)`: true if there is data to be transferred, otherwise false.
- `finished (boolean)`: true if the app execution finished, otherwise false.
- `size (int, optional)`: This value can be used to indicate the size of the data that will be transferred.
 
Example:
`{
  available: true,
  finished: false,
  size: 16
}`

If `available` has been set to `true`, a consecutive `GET /data` call is triggered, fetching the data.

If `finished` has been set to `true` at the coordinator instance, all containers of the workflow are shut down.
 
## GET /data

Using this API call apps can transfer data to the platform.
The response body should contain the data to be transferred. If size was specified in the /status response, the platform will check if the content length matches the size value.
The platform reads the data and redirects in the following way, depending on the sender: 
- if the data is coming from a participant, it will be sent to the coordinator
- if the data is coming from a coordinator, it will be broadcast to all other participants.
 
### POST /data

Using the /data API call the platform transfers data to from the coordinator or broadcasts data from the coordinator to all participants.
The request body should contain the data to be transferred.
The app should handle/consider the received data in the following way, depending on their role:
 - if the receiver is a coordinator, the data is a packet from a participant
 - if the receiver is a participant, the data is a broadcast from the coordinator
 
Besides implementing the above defined API, a federated app optionally can have its own GUI, which is displayed by the FeatureCloud platform.
The GUI is served by the app’s web server, but it’s implementation is fully up to the app developer.

In general, you can develop Feature Cloud apps in any programming language and framework you want, as long as the API is addressed correctly. However, to make it as easy as possible, we already provide different templates shipped with the essential features to efficiently develop an app.

## The Python Template

The Python template already ships a small web server implementing the FeatureCloud API.



