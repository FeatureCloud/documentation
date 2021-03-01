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

### Docker
As a FC app will be a dockerized web server, the whole directory is put into a docker container and the webserver will run in a corresponding python environment. Therefore, the requirements.txt file needs to contain all python pip packages that should be available in the docker container. Files that are not needed in the docker container (sample data, tests, etc.) can be specified in the .dockerignore file. Finally, the app can be dockerized running the build.sh script, after the image name was adapted in the script.

### Webserver
The webserver is executed through the main.py. This happens automatically in the dockercontainer, so in general, a developer does not need to touch the main.py and the files stored in the server_config folder. Only change these files, if you really know what you are doing!

### App algorithm
The app folder stores the files for the actual app execution. The api_ctrl.py implements the FeatureCloud API. Usually, nothing needs to be changed here, as the logic is outsourced to the logic.py.:

```
@api_server.post('/setup')
def ctrl_setup():
    ...


@api_server.get('/status')
def ctrl_status():
    ...


@api_server.route('/data', method='GET')
def ctrl_data_out():
    ...


@api_server.route('/data', method='POST')
def ctrl_data_in():
    ...
```

The api_web.py allows the developers to implement a frontend for the application. The following example shows the current progress of the app:

```
@web_server.route('/')
def index():
    print(f'[WEB] GET /', flush=True)
    return f'Progress: {logic.progress}'
```

The logic.py implements the actual procedure of the federated computation. It already provides the necessary variables:
- `status_available`: Indicates wheter there is data to share. If set to true, make sure data_outgoing is available and contains data.
- `status_finished`: Stops the client if True
- `data_incoming`: Contains the incoming data broadcasted by the coordinator to the participants or sent by the participants to the coordinator
- `data_outgoing`: Contains the data that should be send to the coordinator or broadcasted to the other participants.
- `id`, `coordinator`, `clients`: These variables are set during the setup API call through the `handle_setup()`function. They contain the id of the participant, if this is a coordinator or participant and the overall number of clients participating in the computation 

The following methods handle the data exchange between coordinator and participants, but usually do not need to be changed:
```
def handle_incoming(self, data):
    # This method is called when new data arrives
    self.data_incoming.append(json.load(data))

def handle_outgoing(self):
    # This method is called when data is requested
    self.status_available = False
    return self.data_outgoing
```

The main function, that needs to be adapted by the app developers is the `app_flow()` function. It contains a state machine, and the different states of the app can be defined. The number of each state must be unique, the order does not matter:

```
 # === States ===
state_initializing = 1
state_read_input = 2
state_local_computation = 3
state_wait_for_aggregation = 4
state_global_aggregation = 5
state_finish = 6
```

Now the state machine checks continously in which state the app currently is, and performs the corresponding computation. Note again, that your app needs to support both, coordinator and participants. This is why depending of the clients role, different states are reached. For example, only the coordinator will reach the global computation state, while the participant is waiting in the global gather state at the same time for the global result.

What actually happens in each state and in the app is completely open to the app developer. You can find more examples, such as a Linear Regression (one iteration) or Logistic Regression (multiple iterations) in our GitHub repository: https://github.com/FeatureCloud.

The actual methods of the algorithms are outsourced into the algo.py in this example. In this way, you can simply add methods to the file that can then be used in the federated computation. However, you can also directly write your code into the logic.py files into the corresponding state.

## Publish you App in the AI Store




