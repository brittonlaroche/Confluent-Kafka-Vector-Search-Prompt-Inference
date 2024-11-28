# Step 2. Inference

Basic Layout   

Refer to Step 1. Data Augmentation Vector Embedding github.   
   
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
