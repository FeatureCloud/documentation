# Introduction to FeatureCloud Apps

FeatureCloud’s primary goal is to simplify the development and usage of federated machine learning algorithms. This involves three major challenges: development of apps, distribution of apps, and usage of apps.

FeatureCloud uses Docker as a virtualization technique, to ensure system access of federated apps is as limited as possible. This way FeatureCloud minimizes risks such as leakage of sensitive data or access of data, which should not be included in a machine learning algorithm (e.g. due to lack of consent). 

In order to execute on the FeatureCloud platform, a federated app must implement the FeatureCloud API. The API is designed in a generic way, it puts minimal constraints on the actual implementation of the app, so any kind algorithm can be implemented.

It is important to consider, that the app should be able to act both as coordinator or participant. This is needed because the same app is downloaded by the platform to all sites, and the app's role (coordinator or participant) is determined at setup.

During the run of the app, the coordinator aggregates the data received from participants and broadcasts the aggregated data to all participants. 
The coordinator can act only as aggregator, but can also actively contribute by processing local raw data.

A federated app should act as a web server which is polled by the FeatureCloud platform, so implementing the FeatureCloud API basically means implementing a web server which handles the following requests:

# The FeatureCloud API

## POST /setup
When the participants are ready to start the federated execution (they are connected and prepared the input data) the platform will send the setup request.
This is the starting point of the federated execution, the app can use it as a trigger to start the computation based on it's local data.
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
 
## GET /status
With the response to this request the federated app reports its current status.
The app indicates if there is data to be transferred to the coordinator (participant and controller) or if the execution of the app is finished (coordinator only).
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
- if the data is coming from a participant, it will be redirected to the coordinator
- if the data is coming from a coordinator, it will be broadcast to all other participants.
 
## POST /data
Using this API call the platform transfers data to the app.
The request body should contain the data to be transferred.
The app should handle/consider the received data in the following way, depending on their role:
 - if the receiver is a coordinator, the data is a packet from a participant
 - if the receiver is a participant, the data is a broadcast message from the coordinator
 
Besides implementing the above defined API, a federated optionally can have its own GUI, which is displayed by the FeatureCloud platform.
The GUI is served by the app’s web server, but it’s implementation is fully up to the app developer.

# Templates
In general, you can develop Feature Cloud apps in any programming language and framework you want, as long as the API is addressed correctly. However, to make it as easy as possible, we already provide different templates shipped with the essential features to efficiently develop an app.

# The Python Template
The Python template includes a Flask web server. The main directory contains the following files that might need changes:
- .gitignore: Add files and folders that should not be uploaded to the git repository
- README.md: Describe your app
- build.sh: Used to build your docker image. Here you can decide how your image shall be named
- requirements.txt: Used to install all python packages that are needed for your app. Add all requirements here.

When your app is ready to get tested, run the build.sh to create a docker image that subsequently can be tested in the Feature Cloud Testing Environment.

The actual app development happens in the fc_app, more precisely in the api.py and web.py file. We recommend outsourcing the logic of your algorithm into a separate file, e.g., algorithm_name.py.

# api.py
In the api.py file, the basic API methods (status, data, setup & retrieve_setup_parameters) for Feature Cloud are implemented and can be extended to fulfill your algorithm's requirements. 

All crucial variables should be stored in Redis to be used between the api.py and web.py files. They can be set using the redis_set() method and read using the redis_get() method. Important, predefined variables are:
- 'available': true if data is available for sharing with the coordinator, else false
- 'is_coordinator': true if the client is the coordinator of the analysis., else false
- 'finished': true if the computation is done, else false. After set to true, the server will end the analysis.
- 'nr_clients': Number of clients participating in the analysis.

STEPS should be defined to structure the process of your app. Especially after the setup, different kinds of data need to be exchanged during the analysis. By defining different steps, you can distinguish what data will be exchanged in the data call. You can set the current step using the set_step() method. The get_step() method will give you the current step.

## The data request
The data() method is probably the most crucial in your app development. With the POST request, clients can pull global data from the coordinator, or the coordinator can pull data from the clients.  With the GET request, clients can send data to the coordinator, or the coordinator can broadcast data to the clients. Depending on the step, different data can be exchanged between clients and the coordinator.

## Reading the input
Input files are always located at the directory "/mnt/input/". You can either directly use the files from there (e.g., if there is only one possibility for a file) or implement a file selector in the frontend (web.py).

## Writing the output
All result files need to be saved in the "/mnt/output" directory before the coordinator's finished flag is set. The storage of the results in this directory is essential for the user to download the results or continue with them in the next workflow step. You can also store intermediate results that might be interesting for the users to have.

## Finishing the analysis
As soon as the coordinator computed the final global result, the analysis can be finished. Therefore, the coordinator's finished flag needs to be set (redis_set('finished', true)).

# web.py
In contrast to the api.py, the web.py should contain no app logic itself. It is responsible for the app frontend and has access to all Redis variables set in api.py. It should know about the app's STEPS and show the corresponding frontend for each step. You are entirely free in what you offer in the frontend. Sometimes no frontend is needed at all, but in other cases, a loading screen or even a whole frontend with user input is necessary. 

## Showing different frontends
To show different frontends, use if-else in the root() method to distinguish between the various steps and return the corresponding HTML page stored in the templates folder with return render_template('test.html').

# The Example Mean App
We also provide an extensive example of a Feature Cloud Mean App (https://github.com/FeatureCloud/mean_template). 
