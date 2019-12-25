## Self Managed AWS Alarm Creation

We were working on limiting operational tasks for one of our client. Client has serverless architecture and need to limit all possible impediments while managing his applications.

The client has around 200-300 lambda functions which are exposed via api gateways and dynamo db is the backend database.

In order to create alerts for all these resources, we have created a custom solution which consist of a lambda function, cloud watch event and IAM roles to help the lambda function to trigger trophosphere script.

We are executing cloudwatch event in the interval of a day which mapped to the lambda function. The lambda function has 3.6 python runtime environment and trophoshere script to create/delete lambda/api gateway/dynamodb alerts based on selected metrics.

In order to focus on dynamic resource management via trophoshere , please find the below script and it's description.
```
import sys
import json
from troposphere import Parameter, Ref, Template, Sub
from troposphere.cloudwatch import Alarm, MetricDimension
import boto3
import re
from num2words import num2words #Used to convert numbers to words as alphanumeric allowed in cloudformation.
def alarm_creation(event, context):
    lambda_client = boto3.client('lambda')
    paginator = lambda_client.get_paginator('list_functions')
    response_iterator = paginator.paginate()
    t = Template()
    count = 0
    t.add_description("Dynamic lambda metrics enabled")
    for response in response_iterator:
        functions = response["Functions"]
        for function in functions:
            function_name = function["FunctionName"]
            #Creating pattern which will pick lambda function whose names do not have test/dev
            object = ''.join(c for c in function_name if c.isalpha())
            pattern = re.compile("dev|test")
            if not pattern.search(function_name):
            #Adding resources to template for all selected function
                t.add_resource(
                    Alarm(
                        "resource"+ str(object),
                        AlarmName="resource-"+ str(function_name),
                        Namespace="AWS/Lambda",
                        MetricName="Errors",
                        Dimensions=[
                            MetricDimension(
                                Name="FunctionName"
                                Value=function_name
                            ),
                        ],
                        Statistic="Sum",
                        Period="300",
                        EvaluationPeriods="1",
                        DatapointsToAlarm="1",
                        Threshold="1",
                        Unit="Count",
                        ComparisonOperator="GreaterThanOrEqualToThreshold",
                        AlarmActions=[
                            Sub(
                                "arn:aws:sns:${AWS::Region}:${AWS::AccountId}:devopsincident"
                            ),
                        ],
                    )
                )
                count=count+1
    template = t.to_json()
    cf = boto3.client('cloudformation')
    try:
        response = cf.describe_stacks(
        StackName='lambda-alarm-creation-devops',
        )
    except:
        response = cf.create_stack(
            StackName='lambda-alarm=creation-devops',
            TemplateBody = template
        )
    else:
        response = cf.update_stack(
            StackName='lambda-alarm-creation-devops',
            TemplateBody = template
        )
```
