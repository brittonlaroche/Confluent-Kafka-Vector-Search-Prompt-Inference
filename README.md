# Step 2. Inference

Inference is where we take a users question and create a vector search (semantic Search) against a vector database. We take the results of the vector search as prompts for the LLM.  THen we prompt the LLM with the user's question and the relevant private data returned from the vector search.  When the LLM responds we put the response into a topic to be consumed by the users application.


## Description
Refer to Step 1. Data Augmentation Vector Embedding github.  This github is a continuation of a previous git hub that populated a vector database.  Be sure to check it out as we are usin this data for vector searches in this github example.
[https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding](https://github.com/brittonlaroche/Confluent-Kafka-Vector-Encoding)

Basic Layout   

## Refrence Architecture
We have two different methods of doing a vector search.  One is to use flink SQL and the Federated Search function to perform a search directly againts the vector database using a predifined connection.  The other method is to use the connector architecture to load the user's questions into the vector database and have a trigger on the vector database perform the vector search.  This github will be using the Option with Flink SQL Federated Search.

Using Flink SQL
![Inference Implementation Architecture](/files/img/flinkSqlInferenceArch.png)   

Using the connector Architecture
![Inference Implementation Architecture](/files/img/RefArchitecture.png)   

## Implementation Architecture
![Inference Implementation Architecture](/files/img/InferenceArchitecture.png)    
Create topic user_questions   
Use the Confluent CLI to publish a question to the user questions topic with a guid as key   
Create topic user_questions_vector   
Create flink statement to vector embed user questions into user questions vector   
Create MongoDB Sink Connector to put questions into user questions   
Create MongoDB trigger to perform vector search   
Create a collection with the results of the vector search matching the user query   
Create a MongoDB Source connector to put the search result in the user_prompts topic   
Create Flink statement to prompt the LLM and return responses in JSON format with the guid as the key, insert the results into the user answers topic.  
Use the confluent CLI to consume the answer   
