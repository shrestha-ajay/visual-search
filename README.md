# Visual Search with Amazon SageMaker and AWS DeepLens

This repository provides resources for implementing a visual search engine. Visual search is the central component of an interface where instead of asking for something by voice or text, you *show* what you are looking for.  

Briefly, here’s how the system works. When shown a real world, physical item, a DeepLens device generates a feature vector representing that item. The feature vector generated by the DeepLens device is sent to the AWS cloud using the AWS IoT service. An AWS IoT Rule is used to direct the incoming feature vector from DeepLens to a cloud-based Lambda function, which then uses that feature vector to search for visually similar items in an index of reference item feature vectors. This index is created using SageMaker's k-Nearest Neighbors (k-NN) algorithm, as discussed in Part 1. The search Lambda function returns the top visually similar reference item matches, which are then consumed by a web app via a separate API Lambda function fronted by API Gateway, as shown in the following architecture diagram.  

![Overview](./images/diagram-large.png)

For further explanation of the system architecture, as well as other information about visual search, please refer to the related blog post series.

## How to Access the Model

The model for this project was created with Apache MXNet.  In the notebooks directory of this repository, there is a Jupyter notebook that shows how to create the model and generate feature vectors:  [**Visual Search for AWS DeepLens**](./notebooks/visual-search-feature-generation.ipynb).  If you would rather access an existing, already prepared version of the model, both the MXNet .json model definition and the model parameter file are hosted on Amazon S3.  The S3 bucket location has public read access, and is located at:

```
s3://deeplens-sagemaker-brent/VisualSearch
```

## How to Deploy

In order to deploy this project, you'll need to spin up multiple pieces of infrastructure.  Before doing so, you might want to take a look at the Jupyter notebook in the notebooks directory to gain a deeper understanding of what is happening under the hood:  [**Visual Search notebook**](./notebooks/visual-search-feature-generation.ipynb).

The following instructions cover setup of the minimum number of pieces needed to view search results from feature vectors generated by AWS DeepLens.  The instructions assume you have a basic familiarity with AWS in general and DeepLens in particular.  

1. **DeepLens setup**:  You'll need to create a DeepLens project with (1) the model, and (2) a Lambda function for the DeepLens.
     - **Model**:  Place the model in a S3 bucket with a name of the form ```deeplens-sagemaker-<your_name>```.  Click the **Models** link in the left pane of the DeepLens console, then click **Import Model** and select **Externally trained model.**  Enter the S3 path and model name.
     - **Lambda function**:  From the DeepLens directory of this repository, copy the Lambda function code.  Create a new Lambda function using the blueprint ```greengrass-hello-world``` and paste the copied code into the code editor to completely replace the code in ```greengrassHelloWorld.py```.  Make sure that you save, and then publish the function code by choosing **Publish new version** from the **Actions** menu. 
     - **DeepLens project**:  In the DeepLens console, click **Create new project**, then select **Create new blank project**.  Given the project a name, then add the model and Lambda function you created above.  
     
2.  **Data Store setup**:  This project makes use of two separate data stores.
      - **ElastiCache Redis**:  using the ElastiCache console, create a one-node Redis cluster.  For testing purposes, you can set the instance type and size to ```cache.t2.micro```.  Make a note of the Primary Endpoint URL after creation.  
      - **DynamoDB**:  using the DynamoDB console, create a table named ```VisualSearchFeatures```.  Set the table's Partition key to be a String named **id**.  
     
3.  **API and Search Lambda function setup**:  To create and deploy these Lambda functions, it is recommended to use the AWS Cloud9 IDE for ease of use, and its built-in management capabilities for serverless applications using AWS Serverless Application Model (SAM).
      - **VPC note**:  both Lambda functions must be associated with a VPC because they need to communicate with ElastiCache Redis.  If you are using SAM either with Cloud9 or separately, see the file ```template.yaml``` in this repository for an example of VPC configuration.  Be sure the security group you specify for the Lambda functions is the same as the one for the ElastiCache Redis cluster.  
      - **API Lambda function**:  From the API directory of this repository, copy the Lambda function code.  Create a new Lambda function with a blank template and paste the copied code into the code editor. In the code, change the Redis endpoint URL to the Primary Endpoint URL of your ElastiCache Redis.  The line to be changed is as follows, where you replace the angle brackets and URL inside them:
      ```
      r = redis.StrictRedis(host='<your_Redis_URL>', port=6379, db=0, decode_responses=True)
      ```
      - **Search Lambda function:**  From the Search directory of this repository, copy the Lambda function code.  Create a new Lambda function with a blank template and paste the copied code into the code editor. In the code, change the Redis endpoint URL to the Primary Endpoint URL of your ElastiCache Redis similarly to how you did so for the API Lambda function.  Add an IoT trigger to the Lambda function configuration, with an IoT Rule of the form:
      ```
      SELECT * FROM '<your_DeepLens_device_MQTT_topic>'
      ```
      
4.  **API Gateway setup**:  using the API Gateway console, create an API.  The API only needs one method, a POST method with a resource of the form ```/matches```.  Be sure to enable CORS.  After the API is published, note the API's URL by clicking **Stages**, clicking the Stage name, then copying the **Invoke URL**.

5.  **Front end / web app setup**:  Either download or clone this repository, then in the code replace the API URL with the URL of the API you created in the previous step.  To do this, go to the file ```/app/services/appConfig.js```, then change the line shown below by replacing the angle brackets and URL inside them with your Invoke URL:
```
  .constant('ENV', '<your_API_Gateway_Invoke_URL>');
```
Open the web app code in a text editor that has a captive web server, such as the Brackets editor.  Highlight the index.html file, then launch the web server, which will open a browser window.  In the web app UI, click through the **Visual Search** link. After you add reference item data to DynamoDB (see **Testing** section below), you should see matches populating the UI after a few seconds.


## Testing

In order to view test results in the web app, you'll need to populate the DynamoDB table with some test reference item data.  Some test data is supplied in the test directory of this repository, along with a Python 3 script for adding the data to DynamoDB.  Simply replace the filename ```metadata.json``` in the script with the filename of your test data, then execute the script with the following command in a directory that contains both the script and test data:

```
python3 ./metadata_to_ddb.py
```

Each time you add new reference item data to the DynamoDB table, you will need to restart the Search Lambda function so it picks up the new data.  One easy way to do this is simply re-deploy your existing code without changes.  



# License & Contributing

The contents of this repository are licensed under the [Apache 2.0 License](./LICENSE). 
If you are interested in contributing to this project, please see the [Contributing Guidelines](./contributing/CONTRIBUTING.md).  In connection with contributing, also review the [Code of Conduct](./contributing/CODE_OF_CONDUCT.md).

