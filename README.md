# AWS Lambda Power Tuning

[![Build Status](https://travis-ci.com/alexcasalboni/aws-lambda-power-tuning.svg?branch=master)](https://travis-ci.org/alexcasalboni/aws-lambda-power-tuning)
[![Coverage Status](https://coveralls.io/repos/github/alexcasalboni/aws-lambda-power-tuning/badge.svg)](https://coveralls.io/github/alexcasalboni/aws-lambda-power-tuning)
[![GitHub license](https://img.shields.io/github/license/alexcasalboni/aws-lambda-power-tuning.svg)](https://github.com/alexcasalboni/aws-lambda-power-tuning/blob/master/LICENSE)
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-green.svg)](https://GitHub.com/alexcasalboni/aws-lambda-power-tuning/graphs/commit-activity)
[![GitHub issues](https://img.shields.io/github/issues/alexcasalboni/aws-lambda-power-tuning.svg)](https://github.com/alexcasalboni/aws-lambda-power-tuning/issues)
[![Open Source Love svg2](https://badges.frapsoft.com/os/v2/open-source.svg?v=103)](https://github.com/ellerbrock/open-source-badges/)
[![GitHub stars](https://img.shields.io/github/stars/alexcasalboni/aws-lambda-power-tuning.svg)](https://github.com/alexcasalboni/aws-lambda-power-tuning/stargazers)

AWS Lambda Power Tuning is an AWS Step Functions state machine that helps you optimize your Lambda functions in a data-driven way.

The state machine is designed to be **quick** and **language agnostic**. You can provide **any Lambda function as input** and the state machine will **run it with multiple power configurations (from 128MB to 3GB), analyze execution logs and suggest you the best configuration to minimize cost or maximize performance**.

The input function will be executed in your AWS account - performing real HTTP calls, SDK calls, cold starts, etc. The state machine also supports cross-region invocations and you can enable parallel execution to generate results in just a few seconds. Optionally, you can configure the state machine to automatically optimize the function and the end of its execution.

![state-machine](imgs/state-machine-screenshot.png?raw=true)


Last but not least, the state machine will generate a dynamic visualization of average cost and speed for each power configuration (more details below):

![visualization](imgs/visualization.png?raw=true)


## How to deploy the state machine (AWS Serverless Application Repository)

You can find this app in the [Serverless Application Repository](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:451282441545:applications~aws-lambda-power-tuning) and deploy it with just a few clicks in the AWS Management Console.

You can also integrate the SAR app in your existing CloudFormation stacks - check the `scripts/deploy-sar-app.yml` and `scripts/deploy-sar-app.sh` files for a working reference.

## How to deploy the state machine (AWS SAM)

In case you want to deploy it "manually", you can run `scripts/deploy.sh`.

The script uses the [AWS SAM CLI](https://github.com/awslabs/aws-sam-cli) to create a new CloudFormation stack in your account.

First, install AWS SAM and [configure your AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration):


```bash
$ pip install aws-sam-cli
$ aws configure
```

Now, you can clone this repository as follows:

```bash
$ git clone https://github.com/alexcasalboni/aws-lambda-power-tuning.git
```

Configure your deployment bucket name ([create one first!](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html)) and stack name in the deployment script: 


```bash
# config
BUCKET_NAME=your-sam-templates-bucket
STACK_NAME=lambda-power-tuning
PowerValues='128,512,1024,1536,3008'
```

You can finally deploy the serverless app:

```bash
$ bash scripts/deploy.sh
```


## How to execute the state machine (programmatically)

You can customize the input event in `scripts/sample-execution-input.json` and then run the `scripts/execute.sh` script. It will start a state machine execution, wait for the execution to complete, and then show the execution results.

## How to execute the state machine (web console)

Once the state machine and all the Lambda functions have been deployed, you can execute the state machine and provide an input object.

You will find the new state machine in the [Step Functions Console](https://console.aws.amazon.com/states/) or in your app's `Resources` section.

The state machine name will depend on the stack name (default: `aws-lambda-power-tuning`). Find it and click "**Start execution**". 

Here you can provide the execution input and an execution id (see section below for the full documentation):

```json
{
    "lambdaARN": "your-lambda-function-arn",
    "powerValues": [128, 256, 512, 1024, 2048, 3008],
    "num": 10,
    "payload": {},
    "parallelInvocation": true,
    "strategy": "cost"
}
```

As soon as you click "**Start Execution**" again, you'll be able to visualize the execution.

Once the execution has completed, you will find the execution results in the "**Output**" tab of the "**Execution Details**" section. The output will contain the optimal power configuration and its corresponding average cost per execution.


## State Machine Input

The AWS Step Functions state machine accepts the following parameters:

* **lambdaARN** (required, string): unique identifier of the Lambda function you want to optimize
* **powerValues** (optional, string or list of integers): the list of power values to be tested; if not provided, the default values configured at deploy-time are used (by default: 128MB, 256MB, 512MB, 1024MB, 1536MB, and 3008MB); you can provide any power values between 128MB and 3,008MB in 64 MB increments; if you provide the string `"ALL"` instead of a list, all possible power configurations will be tested
* **num** (required, integer): the # of invocations for each power configuration (minimum 5, recommended: between 10 and 100)
* **payload** (string, object, or list): the static payload that will be used for every invocation (object or string); when using a list, a weighted payload is expected in the shape of `[{"payload": {...}, "weight": X }, {"payload": {...}, "weight": Y }, {"payload": {...}, "weight": Z }]`, where the weights `X`, `Y`, and `Z` are treated as relative weights (not perentages); more details below in the Weighted Payloads section
* **parallelInvocation** (false by default): if true, all the invocations will be executed in parallel (note: depending on the value of `num`, you may experience throttling when setting `parallelInvocation` to true)
* **strategy** (string): it can be `"cost"` or `"speed"` or `"balanced"` (the default value is `"cost"`); if you use `"cost"` the state machine will suggest the cheapest option (disregarding its performance), while if you use `"speed"` the state machine will suggest the fastest option (disregarding its cost). When using `"balanced"` the state machine will choose a compromise between `"cost"` and `"speed"` according to the parameter `"balancedWeight"`
* **balancedWeight** (number between 0.0 and 1.0, by default is 0.5): parameter that express the trade-off between cost and time, 0.0 is equivalent to `"speed"` strategy, 1.0 is equivalent to `"cost"` strategy
* **autoOptimize** (false by default): if `true`, the state machine will apply the optimal configuration at the end of its execution
* **autoOptimizeAlias** (string): if provided - and only if `autoOptimize` if `true`, the state machine will create or update this alias with the new optimal power value


Additionally, you can specify a list of power values at deploy-time in the `PowerValues` CloudFormation parameter. These power values will be used as the default in case no `powerValues` input parameter is provided.

### Usage in CI/CD pipelines

If you want to run the state machine as part of your continuous integration pipeline and automatically fine-tune your functions at every deployment, you can execute it with the script `scripts/execute.sh` (or similar) by providing the following input parameters:

```json
{
    "lambdaARN": "...",
    "num": 10,
    "payload": {},
    "powerValues": [128, 256, 512, ...],
    "autoOptimize": true,
    "autoOptimizeAlias": "prod"
}
```

Of course, you can use different alias names such as `dev`, `test`, `production`, etc.

If you don't configure any alias name, the state machine will only update the `$LATEST` alias.


### Weighted Payloads

Weighted payloads can be used in scenarios where the payload structure and the corresponding performance/speed can vary a lot in production and you'd like to include multiple payloads in the tuning process.

You can use weighted payloads as follows in the execution input:

```json
{
    ...
    "payload": [
        { "payload": {...}, "weight": 5 },
        { "payload": {...}, "weight": 15 },
        { "payload": {...}, "weight": 30 }
    ]
}
```

In the example above the weights `5`, `15` and `30` are used as relative weights. They will correspond to `10%` (5 out of 50), `30%` (15 out of 50), and `60%` (30 out of 50) respectively - meaning that the corresponding payload will be used 10%, 30% and 60% of the time.

For example, if `num=100` the first payload will be used 10 times, the second 30 times, and the third 60 times.

To simplify these calculations, you could use weights that sum up to 100.


## State Machine Output

The state machine will return the following output:

```json
{
  "results": {
    "power": "128",
    "cost": 2.08e-7,
    "duration": 2.9066666666666667,
    "stateMachine": {
      "executionCost": 0.00045,
      "lambdaCost": 0.0005252,
      "visualization": "https://lambda-power-tuning.show/#<encoded_data>"
    }
  }
}
```

More details on each value:

* **results.power**: the optimal power configuration (RAM)
* **results.cost**: the corresponding average cost (per invocation)
* **results.duration**: the corresponding average duration (per invocation)
* **results.stateMachine.executionCost**: the AWS Step Functions cost corresponding to this state machine execution (fixed value for "worst" case)
* **results.stateMachine.lambdaCost**: the AWS Lambda cost corresponding to this state machine execution (depending on `num` and average execution time)
* **results.stateMachine.visualization**: if you visit this autogenerated URL, you will be able to visualize and inspect average statistics about cost and performance; important note: average statistics are NOT shared with the server since all the data is encoded in the URL hash ([example](https://lambda-power-tuning.show/#gAAAAQACAAQABsAL;ZooQR4yvkUa/pQRGRC5zRaADHUVjOftE;QdWhOEMkoziDT5Q4xhiIOMYYiDi6RNc4)), which is available only client-side

## Statistics visualization

The data visualization tool has been built by the community. It is a static website deployed via AWS Amplify Console and it's free to use.

If you don't want to use the visualization tool, you can simply ignore the `stateMachine.visualization` output. No data is ever shared with this tool.

Website repository: [matteo-ronchetti/aws-lambda-power-tuning-ui](https://github.com/matteo-ronchetti/aws-lambda-power-tuning-ui)

Optionally, you could deploy your own custom visualization tool and configure the CloudFormation Parameter named `visualizationURL` with your own URL.

## Security

All the IAM roles used by the state machine adopt the least privilege best practice, meaning that only a minimal set of `Actions` are granted to each Lambda function.

For example, the Executor function can only call `lambda:InvokeFunction`. The Analyzer function doesn't require any permission at all. On the other hand, the Initializer, Cleaner, and Optimizer functions require a broader set of actions.

Although the default resource is `"*"`, you can optionally configure the `lambdaResource` CloudFormation parameter at deploy-time to constrain the IAM permission even more.

For example, you could use a mix of the following:

* Same-region prefix: `arn:aws:lambda:us-east-1:*:function:*`
* Function name prefix: `arn:aws:lambda:*:*:function:my-prefix-*`
* Function name suffix: `arn:aws:lambda:*:*:function:*-dev`
* By account ID: `arn:aws:lambda:*:ACCOUNT_ID:function:*`

## State machine cost

There are three main costs associated with AWS Lambda Power Tuning:

* **AWS Step Functions cost**: it corresponds to the number of state transitions during the state machine execution; this cost depends on the number of tested power values, and it's approximately `0.000025 * (6 + N)` where `N` is the number of power values; for example, if you test the 6 default power values, the state machine cost will be $0.0003
* **AWS Lambda cost** related to your function's executions: it depends on three factors: 1) number of invocations that you configure as input (`num`), the number of tested power configurations (`powerValues`), and the average invocation time of your function; for example, if you test all the default power configurations with `num: 100` and all invocations take less than 100ms, the Lambda cost will be approximately $0.001
* **AWS Lambda cost** related to `Initializer`, `Executor`, `Cleaner`, and `Analyzer`: for most cases it's negligible, especially if you enable `parallelInvocation: true`; this cost is not included in the `results.stateMachine` output to keep the state machine simple and easy to read and debug


## Error handling

If something goes wrong during the initialization or execution states, the `CleanUpOnError` step will be executed. All versions and alises will be deleted as expected (the same happens in the `Cleaner` step).

Note: the error `Lambda.Unknown` corresponds to unhandled errors in Lambda such as out-of-memory errors, function timeouts, and hitting the concurrent Lambda invoke limit. If you encounter it as input of `CleanUpOnError`, it's very likely that the Executor function has timed out and you'll need to enable `parallelInvocation`.

### Retry policy

The executor will retry twice in case any invocation fails. This is helpful in case of execution timeouts or memory errors. You will find the failed execution's stack trace in the `CleanUpOnError` state input.

### How do I know which executor failed and why?

You can inspect the "Execution event history" and look for the corresponding `TaskStateAborted` event type.

Additionally, you can inspect the `CleanUpOnError` state input. Here you will find the stack trace of the error.


## State Machine Internals

The AWS Step Functions state machine is composed of five Lambda functions:

* **initializer**: create N versions and aliases corresponding to the power values provided as input (e.g. 128MB, 256MB, etc.)
* **executor**: execute the given Lambda function `num` times, extract execution time from logs, and compute average cost per invocation
* **cleaner**: delete all the previously generated aliases and versions
* **analyzer**: compute the optimal power value (current logic: lowest average cost per invocation)
* **optimizer**: automatically set the power to its optimal value (only if `autoOptimize` is `true`)

Initializer, cleaner, analyzer, and optimizer are executed only once, while the executor is used by N parallel branches of the state machine (one for each configured power value). By default, the executor will execute the given Lambda function `num` consecutive times, but you can enable parallel invocation by setting `parallelInvocation` to `true`.

Please note that the total invocation time should stay below 300 seconds (5 min), which means that the average duration of your functions should stay below 3 seconds with `num=100`, 30 seconds with `num=10`, and so on. In case you need more time, you can edit the `Timeout` property in the `template.yml` file and redeploy.


## CHANGELOG (SAR versioning)

* *3.2.2*: upgraded runtime to Node.js 12.x
* *3.2.1*: improved scripts and SAR template reference
* *3.2.0*: support for weighted payloads
* *3.1.2*: improved optimal selection when same speed/cost
* *3.1.1*: customizable least-privilege (lambdaResource CFN param)
* *3.1.0*: $LATEST power reset and optional auto-tuning (new Optimizer step)
* *3.0.0*: dynamic parallelism (powerValues as execution parameter)
* *2.1.3*: upgraded runtime to Node.js 10.x
* *2.1.2*: new balanced optimization strategy
* *2.1.1*: custom domain for visualization URL
* *2.1.0*: average statistics visualization (URL in state machine output)
* *2.0.0*: multiple optimization strategies (cost and speed), new output format with AWS Step Functions and AWS Lambda cost
* *1.3.1*: retry policies and failed invocations management
* *1.3.0*: implemented error handling
* *1.2.1*: Node.js refactor and updated IAM permissions (added lambda:UpdateAlias)
* *1.2.0*: updated IAM permissions (least privilege for actions)
* *1.1.1*: updated docs
* *1.1.0*: cross-region invocation support
* *1.0.1*: new README for SAR
* *1.0.0*: AWS SAM refactor (published on SAR)
* *0.0.1*: previous project (serverless framework)

## Contributing
Contributors and PRs are always welcome!

### Tests and coverage

Install dev dependencies with `npm install --dev`. Then run tests with `npm test`, or coverage with `npm run coverage`.
