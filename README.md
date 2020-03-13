# Dining-Concierge-Chatbot

Customer Service is a core service for a lot of businesses around the world and it is getting disrupted at the moment by Natural Language Processing-powered applications.
In this project I implement a serverless, microservice-driven web application. Specifically, I build a Dining Concierge chatbot, that sends you restaurant suggestions given a set of preferences that you provide the chatbot with through conversation.

**The process of building this project as follow:**

* Build and deploy the frontend of the application
  * Implement a chat user interface, where the user can write messages and get responses back.
  * Host frontend in an AWS S3 bucket
    * Set the bucket up for website hosting
    * https://docs.aws.amazon.com/AmazonS3/latest/dev/HostingWebsiteOnS3Setup.html

* Build the API for the application
  * Use API Gateway to setup API
    * use the following API/Swagger specification for my API
      * https://github.com/001000001/aics-columbia-s2018/blob/master/aics-swagger.yaml
      * Use ​http://editor.swagger.io/​ to visualize this file
      * import the Swagger file into API Gateway
        * https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-import-api.html
      * Create a Lambda function (LF0)​ that performs the chat operation
        * Use the request/response model (interfaces) specified in the API specification above
    * For now, just implement a boilerplate response to all messages:
      * ex. User says anything, Bot responds: "I’m still under development. Please come back later."
  * Notes
    * Enable CORS on your API methods
      * https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-cors.html
    * API Gateway can generate an SDK for API, which can use in frontend. It will take care of calling your API, as well as session signing the API calls -- an important security feature
      * https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-generate-sdk-javascript.html
 
* Build a Dining Concierge chatbot using Amazon Lex
  * Create a new bot using the Amazon Lex service. For more information: https://docs.aws.amazon.com/lex/latest/dg/getting-started.html
  * Create a Lambda function (LF1) and use it as a code hook for Lex, which essentially entails the invocation of my Lambda before Lex responds to any of my requests -- this gives us the chance to manipulate and validate parameters as well as format the bot’s responses. More documentation on Lambda code hooks at the following link: https://docs.aws.amazon.com/lex/latest/dg/using-lambda.html
  * Bot functions:
    * Implement the following three intents:
      * GreetingIntent
      * ThankYouIntent
      * DiningSuggestionsIntent
    * The implementation of an intent entails its setup in Amazon Lex as well as handling its response in the Lambda function code hook.
    * For the DiningSuggestionsIntent, I collect the following pieces of information from the user, through conversation:
      * Location
      * Cuisine
      * Dining Time
      * Number of people
      * Phone number
    * Based on the parameters collected from the user, push the information collected from the user (location, cuisine, etc.) to an SQS queue (Q1). More on SQS queues here: https://aws.amazon.com/sqs/, Also confirm to the user that I received their request and that I will notify them over SMS once I have the list of restaurant suggestions.
    
* Integrate the Lex chatbot into my chat API
  * Use the AWS SDK to call my Lex chatbot from the API Lambda (LF0).
  * When the API receives a request, I extract the text message from the API request, and send it to my Lex chatbot, then wait for the response, finally, send back the response from Lex as the API response.
  
* Use the Yelp API to collect 5,000+ random restaurants from Manhattan.
  * Yelp API
    * Get restaurants by self-defined cuisine types
    * Each cuisine type have 1,000 restaurants or so.
    * Make sure restaurants don’t duplicate.
  * DynamoDB (a noSQL database)
    * Create a DynamoDB table and named “yelp-restaurants”
    * Store the restaurants I scrape, in DynamoDB
    * With each item I store, to attach a key to the object named “insertedAtTimestamp” with the value of the time and date of when I inserted the particular record
    * Store those that are necessary for my recommendation.

* Create an ElasticSearch instance using the AWS ElasticSearch Service.
  * Create an ElasticSearch index called “restaurants”
  * Create an ElasticSearch type under the index “restaurants” called “Restaurant”
  * Store partial information for each restaurant scraped in ElasticSearch under the “restaurants” index, where each entry has a “Restaurant” data type. This data type will be of composite type stored as JSON in ElasticSearch.              https://www.elastic.co/guide/en/elasticsearch/guide/current/mapping.html
    * only need to store RestaurantID and Cuisine for each restaurant
    
* Build a suggestions module, that is decoupled from the Lex chatbot.
   * Create a new Lambda function (LF2) that acts as a queue worker. Whenever it is invoked it 1. pulls a message from the SQS queue (Q1), 2. gets a random restaurant recommendation for the cuisine collected through conversation from ElasticSearch and DynamoDB, 3. formats them and 4. sends them over text message to the phone number included in the SQS message, using SNS (https://docs.aws.amazon.com/sns/latest/dg/SMSMessages.html).
    * Use the DynamoDB table “yelp-restaurants” (which you created from Step 1) to fetch more information about the restaurants (restaurant name, address, etc.), since the restaurants stored in ElasticSearch will have only a small subset of fields from each restaurant.
    * Modify the rest of the LF2 function if necessary to send the user text/email.
  * Set up a CloudWatch event trigger that runs every minute and invokes the Lambda function as a result: https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/RunLambdaSchedule.html. This automates the queue worker Lambda to poll and process suggestion requests on its own.
  
  
## Example Interaction 
#### User: Hello ####
Bot: Hi there, how can I help?
#### User: I need some restaurant suggestions.
Bot: Great. I can help you with that. What city or city area are you looking to dine in?
#### User: Manhattan
Bot: Got it, Manhattan. What cuisine would you like to try?
#### User: Japanese
Bot: Ok, how many people are in your party?
#### User: Two
Bot: A few more to go. What date?
#### User: Today
Bot: What time?
#### User: 7 pm, please
Bot: Great. Lastly, I need your phone number so I can send you my findings.
#### User: 123-456-7890
Bot: You’re all set. Expect my suggestions shortly! Have a good day.
#### User: Thank you!
Bot: You’re welcome.

#### A few minutes later, user gets the following text message:

“Hello! Here are my Japanese restaurant suggestions for 2 people, for today at 7 pm: 1. Sushi Nakazawa, located at 23 Commerce St, 2. Jin Ramen, located at 3183 Broadway, 3. Nikko, located at 1280 Amsterdam Ave. Enjoy your meal!”


   
