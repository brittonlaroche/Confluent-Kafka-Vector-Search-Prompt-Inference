# Step 2. Inference

Inference is where we take a users question and create a vector search (semantic Search) against a vector database. We take the results of the vector search as prompts for the LLM.  Then we prompt the LLM with the user's question and the relevant private data returned from the vector search.  When the LLM responds we put the response into a topic to be consumed by the users application.


## Description
Refer to Step 1. Data Augmentation Vector Embedding github.  This github is a continuation of a previous git hub that populated a vector database.  Be sure to check it out as we are using the data generated in step 1 for vector searches in this github example.
[https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding)

Basic Layout   

## Refrence Architecture
We have two different methods of doing a vector search.  One is to use flink SQL and the Federated Search function to perform a search directly againts the vector database using a predifined connection.  The other method is to use the connector architecture to load the user's questions into the vector database and have a trigger on the vector database perform the vector search.  This github will be using the Option with Flink SQL Federated Search.

### Using Flink SQL Federated Search    
This is the direction we will take here in this github.  It has the fewest dependencies and is the easiest to implement:
![Inference Implementation Architecture](/files/img/flinkSqlInferenceArch.png)   

### Using the connector Architecture
The connector Acrhitectire can be used to offload additional work to the Vector Database if you have a situation that calls for complex logic within the vector database.
[Connector example Architecture Here:](/files/img/InferenceArchitecture.png)    

## First Steps 
The setup is relatively easy.  We create some connections to the vector embedding service, the vector database and the LLM.  Then we create some topics user questions, user questions vector, user prompts, and llm Answers.  Then we populate the user questions with data either from the command line or the Web UI.  The next step is to query the vector database with the Flink SQL Federated Search Function with the user questions vector topic data. The restults from the vector search are placed into the user prompts topic.  The final step is to take the users prompts and pass them to the LLM via the Flink SQL ML_Predict function and return the results from the LLM into the LLM answers topic.

For this github will be usingthe OpenAI vector embedding service, MongoDB Atlas for the vector database, and OpenAI for the LLM.  Each of these connections can be created for the Embedding Service, Vector Database and LLM of your choice.  Trying following the examples with these tools as they are all free (aside from OpenAI) and the OpenAI API key is relatively cheap and will last forever on a small one time payment of $10.00.

## Creating the Connections  
We need 3 connections to make inference work with FlinkSQL. We need an embedding connection, a vector database connection and finally a connection to the LLM.  The connections are created in the Confluent CLI. You should issue these commands from the Confluent CLI. If you do not have the Confluent CLI, you can find the installation instructions [here](https://docs.confluent.io/confluent-cli/current/install.html). Instructions for connecting to your environment through the Confluent CLI are available [here](https://docs.confluent.io/confluent-cli/current/connect.html). 

### Vector Embedding Connection   
This is the same procedure used in the first github [https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding)

```
confluent flink connection create openai-vector-connection \
--cloud aws \
--region us-west-2 \
--environment my-env-id \
--type openai \
--endpoint 'https://api.openai.com/v1/embeddings' \
--api-key '<your-openai-api-key>'
```   

### Vector Database Connection   
Run the following command to create a connection resource named “mongodb-connection” that uses your AWS credentials.  The MongoDB API key as shown in API Key Authentication [Atlas Documentation Here](https://www.mongodb.com/docs/atlas/app-services/authentication/api-key/)   

   
```
confluent flink connection create mongodb-connection \
--cloud AWS \
--region us-west-2 \
--type mongodb \
--endpoint <atlas endpoint like mongodb+srv://cluster0.iwuir3o.mongodb.net> \
--api-key <mongodb-api-key> \
--aws-access-key $AWS_ACCESS_KEY_ID \
--aws-secret-key $AWS_SECRET_ACCESS_KEY \
--aws-session-token $AWS_SESSION_TOKEN
```

### LLM Connection   
This connects directly to the OpenAI endpoint for the LLM query
``` 
confluent flink connection create azureopenai-cli-connection \
--cloud aws \
--region us-west-2 \
--environment my-env-id \
--type openai \
--endpoint 'https://api.openai.com/v1/chat/completions' \
--api-key '<your-openai-api-key>'
```
Here is an Example of connecting to the OpenAI endpoint hosted by Microsoft in Azure:   
```
confluent flink connection create azureopenai-cli-connection \
--cloud AZURE \
--region westus2 \
--type azureopenai \
--endpoint https://matrix-central.openai.azure.com/openai/deployments/matrix-central-emb
--api-key <your-azure-api-key>
```

## Create the Topics
We will be running multiple questions, vector searches, LLM responses all together and aysncronously through the applcation we are building.  Lets keep these topics small and clean.  We can always store them elsewhere or keep them around in topics for as long as we like.  For the purposes of this demo lets keep everything nice and tidy.

### Create the topic user_questions   


Use the Confluent CLI to publish a question to the user questions topic with a guid as key   
Create topic user_questions_vector   
Create flink statement to vector embed user questions into user questions vector   
Create a Federated Search using the user's questions  
Create Flink statement to prompt the LLM and return responses in JSON format with the guid as the key, insert the results into the user answers topic.  
Use the confluent CLI to consume the answer   
