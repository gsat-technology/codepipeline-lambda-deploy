# codepipeline-lambda-deploy

#### Dependencies
You need to know how to create Java-based AWS Lambda with Maven. 

#### Summary

This is a CodePipeline project for building and deploying and testing Lambdas. In this case, a Lambda written in Java and built/packaged using Maven.

Why? basically for continuous integration (ci) purposes; this setup allows you to test/build/deploy and then test out the deployed version.

Would also be useful when prototyping to get lambdas up and working that you can actually use and keep on applying changes. 

CodePipeline steps are:
1. Source from Github
2. Build using Codebuild. Creates both a java deployment package and CloudFormation package
3. CloudFormation: create change set
4. CloudFormation: execute change set 
5. Test using CodeBuild

Once the steps are completed, the Lambda running and can be executed.

Parameters:

| Name                      	| Purpose                                                                                                	|
|---------------------------	|--------------------------------------------------------------------------------------------------------	|
| GithubOwner               	| Owner (username) of the Github account                                                                 	|
| GithubRepoName            	| name of the repo that will be checked-out and built                                                    	|
| GithubPersonalAccessToken 	| Github API token used by CodePipeline to detect repo changes (see section below on how to create this) 	|
| TestScript                    | Path to script in repo which runs tests against lambda                                                    |


#### CloudFormation template

You need this CloudFormation template to sit with your source code in the root dir of the project.

Note the `<output_jar_name>`. You need to replace this with what ever your jar name will be once `maven package` is run.

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: 'AWS::Serverless-2016-10-31'
Resources:
  MyLambda:
    Type: 'AWS::Serverless::Function'
    Properties:
      Handler: my.lambda.Handler::handleRequest
      Runtime: java8
      CodeUri: ./target/<output_jar_name>.jar
```

The above template is what CodePipeline will deploy i.e. it just creates a lambda resource with the built lambda deployment package that gets created in the CodeBuild step

#### Tests

The final step is the execution of a shell script. The script is executed with a single parameter (the name of the lambda). Your script should be written so that tests which fail result in a non-0 exit.

The CodeBuild buildspec installs the `jq` tool before running the script which may or may not be useful to you (depends on how you want to write your test)

e.g. it could be as simple as this:

```
#!/bin/bash

lambda=$1

aws lambda invoke --function-name $lambda --payload "{\"test\": \"payload\"}" result.txt
result=$(cat result.txt)

if [ "$result" = $someExpectedOutcome ]
then
    echo "success"
else
    echo "failed"
    exit 1
fi
```

#### Obtain a Github Personal Access Token 
1. Sign into Github 
2. Settings -> Developer Settings -> Personal Access Tokens
3. click 'Generate New Token'
4. Give some description like 'CodePipeline' 
5. tick all boxes in 'repo' and 'admin:repo_hook' sections 
6. Copy the personal access token

#### Issues
When tearing down the CodePipeline stack, ensure that you tear down the deployed stack (i.e. the stack with the lambda) first. Otherwise the CodePipeline stack cannot be torn down (easily)