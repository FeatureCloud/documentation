# Demo

This guide describes how the workflow described in https://arxiv.org/abs/2105.05734 can be executed.

Make sure you have followed the [installation](installation.md) steps so that you have an account and running 
controller.

To see how a workflow can be accomplished in FeatureCloud, also have a look at the [website manual](https://featurecloud.ai/manual).

## Steps

**1. Add apps to your library**

From the AI store, select the 
[cross validation app](https://featurecloud.ai/ai-store/52), 
[normalization app](https://featurecloud.ai/ai-store/45), 
[random forest app](https://featurecloud.ai/ai-store/50) and 
[classification evaluation app](https://featurecloud.ai/ai-store/48).

You need to add them to your library by pressing the 'Add to library' button.

**2. Create project and workflow**

After adding the apps to your library, you can use them in your projects. Go to the project page and create a new 
project by pressing 'Create project'. After creation, go to the project page and add the apps in the order mentioned 
above and press 'Finalize'.

**3. Invite other participants (optional)**

The purpose of a federated workflow is collaboration with other participants who have additional data. Therefore, in
practice, you will always want to invite other participants. The apps used in this demo can also operate alone, 
therefore this step is optional.

To invite other participants, you need to create so-called tokens. They are random strings and uniquely identify a 
project and can only be used once. Create one for each participant by pressing 'Token' and send it to them. They can 
join this project by pasting the token on the join project page. All participants must have a user account and the 
FeatureCloud controller running.

Once all participants have joined, or no additional participants are required, press 'Start'.

**4. Hand over data**

The workflow described in this demo requires a config file and data. You can find both in the `data/<1-5>` directories.
We provide 5 splits of the ILPD dataset (see manuscript). You can use all of them (one per participant) or a subset 
thereof. In case you are running the workflow with a single participant, you can use any one of the splits.

Click the 'Add file...' button and first add the `config.yml` file and then the `data.csv` file from one of the splits 
(e.g., `data/1/config.yml` and `data/1/data.csv`).

After you have added both files, click 'Upload'.

**5. Monitor the workflow**

After uploading the data the workflow will start. From now on, no user interaction is required until the workflow has
finished. Each step in the workflow will produce intermediate data, which you can download. The last step will produce
then final results, including a plot displaying the performance.

The results obtained for a single participant using data split 1 can be found in the folder `data/results/1`.

### Having issues using FeatureCloud?

If you encounter a problem, don't hesitate to raise an issue in this repository.