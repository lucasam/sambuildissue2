# Nested Application from the Local File System build does not work properly

Issue reported https://github.com/awslabs/aws-sam-cli/issues/1470

## Issue description

According the documentation 
https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-template-nested-applications.html

The `AWS::Serverless::Application` can point to another template file (check Defining a Nested Application from the Local File System)

if you trigger the commands sam build; sam package, sam deploy the result of the nested application is a broken zip file.

## Scenario
```
/ - javaproject
  | - template.yaml
  | - pom.xml
  | - src/main/java...
/ - template.yaml  (parent)
```
Parent template references javaproject/template.yaml as AWS::Serverless::Application`
```yaml
Resources:
  AvatarApplication:
    Type: AWS::Serverless::Application
    Properties:
      Location: javaproject/template.yaml
```

The sequence `sam build; sam package; sam deploy ` when triggered from parent directory completes. But the zipfile for the function under the nested application is wrongly built.


Let's check step by step using the sample code https://github.com/lucasam/sambuildissue2

Cloning sample code
````bash
git clone https://github.com/lucasam/sambuildissue2
cd sambuildissue2
````

`sam build`

```bash

lucasam$ .sam build --debug 
Using SAM Template at /Users/lucasam/Documents/src/sambuildissue2/template.yaml
Telemetry endpoint configured to be https://aws-serverless-tools-telemetry.us-west-2.amazonaws.com/metrics
'build' command is called
No Parameters detected in the template
1 resources found in the template

Build Succeeded

Built Artifacts  : .aws-sam/build
Built Template   : .aws-sam/build/template.yaml

Commands you can use next
=========================
[*] Invoke Function: sam local invoke
[*] Package: sam package --s3-bucket <yourbucket>
    
Sending Telemetry: {'metrics': [{'commandRun': {'awsProfileProvided': False, 'debugFlagProvided': True, 'region': '', 'commandName': 'sam build', 'duration': 21, 'exitReason': 'success', 'exitCode': 0, 'requestId': '7178268e-8384-4ad2-a044-cfd938769ca7', 'installationId': '893c5aa8-d107-4b3b-8f82-e7cb5c9ce2fe', 'sessionId': '907647c9-698e-428a-a1ca-d86ca056821e', 'executionEnvironment': 'CLI', 'pyversion': '3.7.4', 'samcliVersion': '0.22.0'}}]}
HTTPSConnectionPool(host='aws-serverless-tools-telemetry.us-west-2.amazonaws.com', port=443): Read timed out. (read timeout=0.1)
```
`sam package`
```bash
lucasam$ sam package --output-template-file packaged.yaml  --s3-bucket $BUCKET 
Uploading to 41c665fe10f20b0417eb7f684d7a8072.template  791 / 791.0  (100.00%)
Successfully packaged artifacts and wrote output template to file packaged.yaml.
Execute the following command to deploy the packaged template
aws cloudformation deploy --template-file /Users/lucasam/Documents/src/sambuildissue2/packaged.yaml --stack-name <YOUR STACK NAME>
```

`sam deploy`
```bash
lucasam$ sam deploy  --template-file packaged.yaml  --stack-name sam-bug  --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
    --region us-east-1
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - sam-bug
````


Everything looks fine so far. *BUT*

If you inspect the nested application you will find

````bash
f01898820a1c:x2 lucasam$ aws s3 cp s3://<BUCKET>/261b18ebec06a0f13ad2709228e60cca . 
download: s3://<BUCKET>/261b18ebec06a0f13ad2709228e60cca to ./261b18ebec06a0f13ad2709228e60cca
f01898820a1c:x2 lucasam$ unzip 261b18ebec06a0f13ad2709228e60cca
Archive:  261b18ebec06a0f13ad2709228e60cca
  inflating: template.yaml           
  inflating: sam-issue2.iml          
  inflating: pom.xml                 
  inflating: Readme.md               
  inflating: License.txt             
  inflating: src/main/java/com/test/FunctionOne.java  
````

## Issue
- Observe that the zip file representing the nested application is just a zip of the Nested application folder.
- Expected. SAM should interpret template.yaml of the nested application and build it properly.

## Workaround
- Build use `sam build` and `sam package` at the nested application. Then point at parent template the `packaged.yaml` instead of `template.yaml`