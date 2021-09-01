# App development

FeatureCloud uses Docker as a virtualization technique to ensure system access of federated apps is as limited as
possible. This way FeatureCloud minimizes risks such as leakage of sensitive data or access of data, which should not
be included in a machine learning algorithm (e.g. due to lack of consent).

In order to be executable on the FeatureCloud platform, a federated app must implement the FeatureCloud API. The API is
designed in a generic way, it puts minimal constraints on the actual implementation of the app, therefore many kinds
algorithm can be implemented.

In FeatureCloud, there are two roles involved in federated learning: the participant and coordinator. Participants have
direct access to the data and usually use it to train a local model (e.g. neural network). The coordinator then collects
all the submodels and aggregates them into one model. Possibly, there are multiple iterations involved. In that case,
the coordinator sends the model back to the participants who continue improving it individually and send it back to
the coordinator and so forth, until convergence is reached.

Apps need to be implemented in a way such that they can assume either role dynamically after startup. This is needed
because the same app is downloaded by the platform to all sites, and the app's role (coordinator or participant) is
determined at setup. Therefore, a coordinator is a special participant, that receives the local results of all
participants and broadcasts the aggregated results back.

## The FeatureCloud API

A federated app should act as a web server which is polled regularly by the FeatureCloud platform and responds to the 
FeatureCloud API. It must provide the following requests, that are already integrated in the FeatureCoud app templates.

A detailed description of the API (OpenAPI 3.0.1) can be found here: 
https://featurecloud.ai/assets/api/redoc-static.html

## The Python Template

The Python template already ships a small web server implementing the FeatureCloud API.

### Docker

As a FeatureCloud app will be a dockerized web server, the whole directory is put into a docker container and the 
webserver will run in a corresponding python environment. Therefore, the requirements.txt file needs to contain all python pip packages that should be available in the docker container. Files that are not needed in the docker container (sample data, tests, etc.) can be specified in the .dockerignore file. Finally, the app can be dockerized running the build.sh script, after the image name was adapted in the script.

### Webserver
The webserver is executed through the `main.py`. This happens automatically in the docker container, so in general, a 
developer does not need to touch the `main.py` and the files stored in the server_config folder. Only change these files, if you really know what you are doing!

### App algorithm
The app folder stores the files for the actual app execution. The `api_ctrl.py` implements the FeatureCloud API. 
Usually, nothing needs to be changed here, as the logic is outsourced to the `logic.py`:

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

The `api_web.py` allows the developers to implement a frontend for the application. The following example shows the 
current progress of the app:

```
@web_server.route('/')
def index():
    print(f'[WEB] GET /', flush=True)
    return f'Progress: {logic.progress}'
```

The `logic.py` implements the actual procedure of the federated computation. It already provides the necessary variables:
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

The main function, that needs to be adapted by the app developers is the `app_flow()` function. It contains a state 
machine where the different states of the app can be defined. The number of each state must be unique, the order does 
not matter:

```
 # === States ===
state_initializing = 1
state_read_input = 2
state_local_computation = 3
state_wait_for_aggregation = 4
state_global_aggregation = 5
state_finish = 6
```

Now the state machine checks continously in which state the app currently is, and performs the corresponding 
computation. Note again, that your app needs to support both, coordinator and participants. This is why depending of 
the clients role, different states are reached. For example, only the coordinator will reach the global computation 
state, while the participant is waiting in the global gather state at the same time for the global result.

What actually happens in each state and in the app is completely open to the app developer. You can find more examples, 
such as a Linear Regression (one iteration) or Logistic Regression (multiple iterations) in our GitHub repository: 
https://github.com/FeatureCloud.

The actual methods of the algorithms are outsourced into the `algo.py` in this example. This way, you can simply add 
methods to the file that can then be used in the federated computation. However, you can also directly write your code 
into the `logic.py` files into the corresponding state.

## Publish your app in the AI Store

1. Login to your FeatureCloud account
2. Make sure that you have the developer tools enabled in your account
3. Go to the AI store and click Create App in the developer section
4. Fill in the app details and choose an image name
5. Build your docker image and tag it with `registry.featurecloud.eu:5000/image_name`, where `image_name` is the name 
   you chose previously
6. Your app should now appear in the app store as uncertified app and can be used by other users
