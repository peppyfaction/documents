## Self Managed AWS Alarm Creation

We were working on limiting operational tasks for one of our client. Client has serverless architecture and need to limit all possible impediments while managing his applications.

The client has around 200-300 lambda functions which are exposed via api gateways and dynamo db is the backend database.

In order to create alerts for all these resources, we have created a custom solution which consist of a lambda function, cloud watch event and IAM roles to help the lambda function to trigger trophosphere script.

We are executing cloudwatch event in the interval of a day which mapped to the lambda function. The lambda function has 3.6 python runtime environment and trophoshere script to create/delete lambda/api gateway/dynamodb alerts based on selected metrics.

In order to focus on dynamic resource management via trophoshere , please find the below script and it's description.

![embedded code](https://gist.github.com/peppyfaction/778bd4fd622dcb730d70a82b8f443e2e)
