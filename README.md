# Step 2. Inference

Inference is where we take a users question and create a vector search (semantic Search) against a vector database. We take the results of the vector search as prompts for the LLM.  Then we prompt the LLM with the user's question and the relevant private data returned from the vector search.  When the LLM responds we put the response into a topic to be consumed by the users application.   
   
![Inference Implementation Architecture](/files/img/inferenceGeneralPattern.png)  
   
## 4 Steps to Building a GenAI Application
There are 4 steps to building a GenAI Application and I have included a github for each step.    
The githubs (some still work in progress) are indexed here:   
Step 1. Data Augmenatation: [Confluent-Kafka-Vector-Encoding](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding)   
Step 2. Inference: [Confluent-Kafka-Vector-Search-Prompt-Inference](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Search-Prompt-Inference)   
Step 3. Workflows: [Confluent-Kafka-Vector-Search-Workflows](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Search-Workflows)   
Step 4. Post Processing: [Confluent-Kafka-Vector-Search-Post-Processing](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Search-Post-Processing)   
   
## Inference Description
This github explores the second step in Building a RAG Enabaled Gen AI application.  Inference is where we query the vector database and marry our private data with the user question to prompt the LLM with the data it needs to make an appropriate response.  
   
This github is a continuation of a previous github that populated a vector database.  Be sure to check it out as we are using the data generated in step 1 for vector searches in this github example. [Confluent-Kafka-Vector-Encoding](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding)


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
If you completed the first step then you should already have this connection. Its listed here for the sake of completeness.   
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

The Atlas endpoint resembles mongodb+srv://cluster0.iwuir3o.mongodb.net      
```
confluent flink connection create mongodb-connection \
  --cloud aws \
  --region westus2 \
  --type mongodb \
  --endpoint ${atlas_endpoint} \
  --username ${atlas_username} \
  --password ${atlas_password}
```

### LLM Connection   
This connects directly to the OpenAI endpoint for the LLM query
``` 
confluent flink connection create openai-llm-connection \
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
Navigate to the topics menu item inside your cluster and press the "New Topic" button.  Enter "user_questions" for the topic name. Set the number of partitions to 1. Click the "Show Advanced Settings" Link. Set the retention time to 1 hour and the retension size to 1GB.  Then click save and create. We are keepingthe topics small for the demo, its not necessary but a good exercise when you are using a free tier basic cluster for a demo.  
![User Questions Topic](/files/img/userQuestionsTopic.png)   

Notice we are not setting a schema or data contract just yet.  There is a reason for this as we will see how to modify the topics and schemas for our needs in a future section.

## Vector Embed the User Questions

Lets take a moment to undertsand the vector search.  All of our product data has been converted into vectors.  Now our user is going to ask a question and we need to search based on meaning.  In order to get the meaning out of the user's question we need to vector embed the question.  Lets break this down with a curl command.  Lets say we have a user ask the following three questions or commands if you will.
   
"Find me a pair of mens formal shoes in medium size"   
"Show me little girl shoes in medium size"    
"Show me several options of a cheap read summer dress in medium size"    
    
    
We can take this user question and call the vector embedding service to get the vector back directly with a curl command.  Its a good exercize to see what we are about to do next in Flink SQL  Try the following, lets export your openAI api Key as a session variable and use it in the curl command.

```
export OPENAI_API_KEY="<your-api-key-here>"
```

```
curl https://api.openai.com/v1/embeddings \
 -H "Content-Type: application/json" \
 -H "Authorization: Bearer $OPENAI_API_KEY" \
 -d '{ "input": "Find me a pair of mens formal shoes in medium size", "model": "text-embedding-3-small" }'
```



Take a look at some of the results of the curl commands.  [Sample Vector Searches](/files/sampleVectorSearches)   
You can capture your own by redirecting the output to a text file.  For example:

```
curl https://api.openai.com/v1/embeddings \
 -H "Content-Type: application/json" \
 -H "Authorization: Bearer $OPENAI_API_KEY" \
 -d '{ "input": "Show me little girl shoes in medium size" }' > test.txt
```

### Publish a question to the user_questions topic
There are a few things we can do in JSON to speed things up nicely.  Tyoically you can define diffent roles (user, system, assistant)  and pass in content as prompts.  For example:   
```
{"role": "user", "content": "Find me a pair of mens formal shoes in medium size."}
```
   
To tell the LLM how to respond or to provide product prompts we can use the system role.   
```
{"role": "system", "content": "Please respond with a JSON document that has fields for product_id, store_id and price."}
```
   
The responses from the LLM typically come back with the role of assistant.   
   
Lets publish the users question in the cloud UI and see what happens.  Open the "user_questions" topic and click on the "messages" tab.  Publish a new message with the users question from above. Use a key like 8888.    

![Add A question ](/files/img/add_user_questions.png)   
Now we should see a message in the user_questions topic

![See the new Question](/files/img/user_questions_message.png)   
   
Now lets query this in flink and get the Vector for the users question.   

If we have not already done so in the first github we need to create the following model function to call the vector embedding service.  Log into FlinkSQL and issue the following command. 
```
confluent flink shell --compute-pool lfcp-pool-from-gui --environment env-myenv-from-gui
```
And then
```
CREATE MODEL `vector_encoding`
INPUT (input STRING)
OUTPUT (vector ARRAY<FLOAT>)
WITH(
  'TASK' = 'embedding',
  'PROVIDER' = 'openai',
  'OPENAI.CONNECTION' = 'openai-vector-connection'
);
```

### Query the user_questions topic in FlinkSQL
Opne the FlinkSQL WebUI "SQL Workspace" from the environment level or do it via command line:

```
Select * from `user_questions`;
```
![FinkSQL User Question](/files/img/flinkSQLUglyUserQuestions.png)   
    
Wow that looks horrible! What the heck is that?  Where is the user question?  Don't panic.  This happens all the time.  New developers add data to topics with out knowing what the schema registry is.  Without a schema FlinkSQL can't process the user questions. Luckily we now process schemaless data in FlinkSQL! [https://docs.confluent.io/cloud/current/flink/how-to-guides/process-schemaless-events.html](https://docs.confluent.io/cloud/current/flink/how-to-guides/process-schemaless-events.html)   

How do you process schemaless events?  Its simple you ad a schema after the fact.  No, really.  That is how your process schemaless data in FlinkSQL, you add a schema. Lets do that now.  Go to the user_questions topic and add a data contract.  Then click the schema tab, add a new schema, make sure it is of type JSON. Add in the following Schema that defines the role and content.  We don't have to use this schema we can set any we like.  I use this schema because its easy, simple and yet powerful, and I like it.

```
{
  "additionalProperties": false,
  "description": "user_questions schema.",
  "properties": {
    "content": {
      "description": "The content provided by the role.",
      "type": "string"
    },
    "role": {
      "description": "The role for the content: user, system, assistant.",
      "type": "string"
    }
  },
  "title": "UserQuestion Record",
  "type": "object"
}
```
   
Now lets query it again   
   
```
Select * from `user_questions`;
```
![FinkSQL User Question](/files/img/userQuestionsReadable.png)   
Much Better! We are ready to vector encode it!   

### Insert the new user question vector 
This final step for vector encoding the users questions should be rather easy. We simply call the vector encoding funcion we created earlier and select everything from the user_questions topic. We just need a table or topic to insert into.

```
CREATE TABLE `user_questions_vector` (                       
    `role`         STRING,                      
    `content`      STRING,                      
    `vector`      ARRAY<FLOAT>
) WITH (
  'value.format' = 'json-registry'
);
```  
By creating the table in FlinkSQL and defining the data types we automatically create the topic and schema to go with it.
  
```
insert into `user_questions_vector` select role, content, vector from `user_questions`,
lateral table (ml_predict('vector_encoding', content));
```
We can see the statement running by looking at the FLinkSQL runing statements.
![FinkSQL Running](/files/img/flinkSQLRunning.png)   
   
   
We can see the vector created for the user's question by looking at the users_questions_vector topic.   

      
![FinkSQL Running](/files/img/userQuestionsVector.png)   

## Lets do a Vector Search!
We are now ready to perform a vector search against the vector database with our new vector field in the user_questions_vector topic.  To do this we will connect to our MongoDB Atlas instance. We will be using the MongoDB connection we created earlier:
   
The Atlas endpoint resembles mongodb+srv://cluster0.iwuir3o.mongodb.net      
```
confluent flink connection create mongodb-connection \
  --cloud aws \
  --region westus2 \
  --type mongodb \
  --endpoint ${atlas_endpoint} \
  --username ${atlas_username} \
  --password ${atlas_password}
```




Create topic user_questions_vector   
Create flink statement to vector embed user questions into user questions vector   
Create a Federated Search using the user's questions  
Create Flink statement to prompt the LLM and return responses in JSON format with the guid as the key, insert the results into the user answers topic.  
Use the confluent CLI to consume the answer   
