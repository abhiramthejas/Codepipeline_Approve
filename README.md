# Codepipeline_Approve

In this project, we are building an AWS CodePipeline which allows a user to deploy code only at a specified time using AWS EventBridge and Lambda.

Before creating the CodePipeline, we need to make sure to create the CodeDeployment and deployment group under AWS CodeDeploy.

## Code-Deploy

To create the CodeDeploy application:

   1. Choose AWS > CodeDeploy > Create Application.
   2. Provide an application name and choose a compute platform. (I am using EC2 as my compute platform).
   3. Click "Create".

After creating the application, choose that application and create a deployment group. To do this:

   1. Click "Create deployment group" under the CodeDeploy application you just created.
   2. Provide a group name, service role, and deployment type (blue green / In place. I am using an example of In-place).
   3. Under the step "Environment configuration", choose which environment you need to deploy the code. You can select EC2, Auto Scaling group, and On-  premises instances.
   4. If you are selecting EC2, make sure to give proper tags in the settings to choose the correct instance and create the deployment group.

After creating the CodeDeploy, we can start creating the CodePipeline.

## Code-pipeline

To create an AWS CodePipeline:

    Choose the menu AWS > CodePipeline > Create Pipeline.
    Provide a pipeline name and role name.
    On the second step of the pipeline creation, choose the provider where your code is committed. (I am using Github as my source provider. Make sure our AWS account is connected with Github).
    In the third stage of the pipeline, provide your CodeBuild stage. (I am skipping this option as I am directly pushing the code to the EC2 instance using CodeDeploy).
    On the fourth step, provide the deploy provider. (I am using CodeDeploy as my deploy provider and choosing my application name and deployment group name).
    At the last step, review your settings and create the CodePipeline.

Next, we need to add our Approve stage in the CodePipeline by selecting your CodePipeline and clicking "Edit". Add a stage after the source and give it a stage name like "Approval-stage" and click on "Add action group" to add the action in the stage.

To add an action group:

   1. Action name (provide a name). (I used "Manual Approve").
   2. Choose Action provider as Manual approve.
   3. Click "Done".

We need to add one more manual approve stage after this stage and we will use Lambda to trigger it.

To create the manual approve stage:

    1. Follow the same steps as before.
    2. Use "Action name" and "Stage name" as "Lambda-Approve".

![image](https://user-images.githubusercontent.com/17767960/209428051-dbf105a3-a2fd-4a1a-9a7b-04104d963bab.png)


## Lamda Function

To create a Lambda function:

   1. Choose "Create function".
   2. Give a function name and select the runtime as Python 3.8.
   3.  Make sure the Lambda function you created has the role to connect with the CodePipeline by adding pipeline read access.

Add the following code to the Lambda function:

```
import boto3
import datetime as dt

def lambda_handler(event, context):
# Create a client for the CodePipeline service
    codepipeline = boto3.client('codepipeline')

    pipeline_name = 'your pipeline name'
    response = codepipeline.get_pipeline_state(name=pipeline_name)
    token = response['stageStates'][2]['actionStates'][0]['latestExecution']['token']
    current_date = dt.datetime.now()



# The name of the stage containing the manual approval
    stage_name = 'your stage name'
    action_name = 'your action name'

# Approve the job
    response = codepipeline.put_approval_result(
        pipelineName=pipeline_name,
        stageName=stage_name,
        actionName=action_name,
        result={
         'summary': f"Stage approved at {current_date}",
         'status': 'Approved'
        },
        token=token
    )
    
    print(response)
    return "Success"
```
The codepipeline client is used to access the CodePipeline service. The get_pipeline_state method is called to retrieve information about the current state of a pipeline. The put_approval_result method is then used to approve a job in the pipeline.

The stage_name and action_name variables contain the name of the stage and action in the pipeline where the manual approval is located. The current_date variable stores the current date and time. This information is used to create a summary message which is included in the result dictionary that is passed to the put_approval_result method.

The line of code token = response['stageStates'][2]['actionStates'][0]['latestExecution']['token'] is extracting the token value from the action object and assigning it to the token variable. The indices 2 and 0 are used to access the third stage and the first action in the pipeline.

The stage_name and action_name variables contain the names of the stage and action in the pipeline where the manual approval is located. The current_date variable stores the current date and time. This information is used to create a summary message which is included in the result dictionary that is passed to the put_approval_result method.

## Event Bridge

To create the event-bridge rule choose AWS > Event-Bridge > Create rule 

Add rule name and choose option Schedule as the rule type give your required time in the "Cron expression".

![image](https://user-images.githubusercontent.com/17767960/209428110-9108e82a-aa49-4f59-a473-95a72bf2b24c.png)


In the third step to choose the target for the Eventbridge choose AWS Service as Target types and Target as Lambda with your Function name add proper tagging and create the rule.

Finally the Manual Approve will get execute at the scheduled Time !!



