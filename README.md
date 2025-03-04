## AWS for Fluent Bit Docker Image

### Contents

- [Versioning FAQ](#versioning-faq)
- [Debugging Guide](troubleshooting/debugging.md)
- [Public Images](#public-images)
    - [Using the stable tag](#using-the-stable-tag)
    - [Using SSM to find available versions](#using-ssm-to-find-available-versions)
    - [Using SSM Parameters in CloudFormation Templates](#using-ssm-parameters-in-cloudFormation-templates)
    - [Using image tags](#using-image-tags)
        - [Amazon ECR Public Gallery](#amazon-ecr-public-gallery)
        - [Docker Hub](#docker-hub)
        - [Amazon ECR](#amazon-ecr)
- [Plugins](#plugins)
- [Using the AWS Plugins outside of a container](#using-the-aws-plugins-outside-of-a-container)
- [Development](#development)
    - [Releasing](#releasing)
    - [Developing Features in the AWS Plugins](#developing-features-in-the-aws-plugins)
- [Fluent Bit Examples](#fluent-bit-examples)
- [License](#license)


### Versioning FAQ

The version of the AWS for Fluent Bit image is not linked to the version of Fluent Bit which it contains.

**What does the version number signify?**

We use the standard `major.minor.patch` versioning scheme for our image, AKA Semantic Versioning. The initial release with this versioning scheme is `2.0.0`. An update to the patch version indicates backwards compatible bug fixes, a minor version change indicates new backwards compatible functionality, and a major version change indicates backwards incompatible changes.

The AWS for Fluent Bit image includes the following contents:
* A base linux image (currently Amazon Linux)
* Fluent Bit
* Several Fluent Bit Go Plugins

A change in any one of these pieces would lead to a change in our version number.

**What about the 1.x image tags in your repositories?**

The AWS for Fluent Bit image was launched in July 2019. Between July and October of 2019 we simply versioned the image based on the version of Fluent Bit that it contained. During this time we released `1.2.0`, `1.2.2` and `1.3.2`.

The old versioning scheme was simple and it made it clear which version of Fluent Bit our image contained. However, it had a serious problem- how could we signify that we had changed the other parts of the image? If we did not update Fluent Bit, but updated one of the plugins, how would we signify this in a new release? There was no answer- we could only release an update when Fluent Bit released a new version. We ultimately realized this was unacceptable- bug fixes or new features in our plugins should not be tied to the Fluent Bit release cadence.

Thus, we moved to the a new versioning scheme. Because customers already are relying on the `1.x` tags, we have left them in our repositories. The first version with the new scheme is `2.0.0`. From now on we will follow semantic versioning- but the move from `1.3.2` did not follow semantic versioning. There are no backwards incompatible changes between `aws-for-fluent-bit:1.3.2` and `aws-for-fluent-bit:2.0.0`. Our release notes for `2.0.0` clearly explain the change.

**Does this mean you are diverging from fluent/fluent-bit?**

No. We continue to consume Fluent Bit from its main repository. We are not forking Fluent Bit.

### Debugging Guide

[Please read the debugging.md](troubleshooting/debugging.md)

### Public Images

Each release updates the `latest` tag and adds a tag for the version of the image. The `stable` tag is also available which marks a release as the latest stable version. Deploying `latest` to prod without going through a test stage first is not recommended.

#### Using the stable tag

A `stable` tag can be trusted that it is the latest version in which there are no high impact bugs in Fluent Bit. A release may be marked as stable if the following rules are all met:
* It has been out for at least 2 weeks or is a CVE patch with no Fluent Bit changes
* No bugs have been reported in Fluent Bit which we expect will have high impact for AWS customers. This means bugs in the components that are most frequently used by AWS customers, such as the AWS outputs of the tail input

#### Using SSM to find available versions

As of 2.0.0, there are SSM Public Parameters which allow you to see available versions. These parameters are available in every region that the image is available in. Any AWS account can query these parameters.

To see a list of available version tags, run the following command:

```
aws ssm get-parameters-by-path --path /aws/service/aws-for-fluent-bit/ --query 'Parameters[*].Name'
```

Example output:

```
[
    "/aws/service/aws-for-fluent-bit/latest"
    "/aws/service/aws-for-fluent-bit/2.0.0"
]
```

To see the ECR repository ID for a given image tag, run the following:

```
$ aws ssm get-parameter --name /aws/service/aws-for-fluent-bit/2.0.0
{
    "Parameter": {
        "Name": "/aws/service/aws-for-fluent-bit/2.0.0",
        "Type": "String",
        "Value": "906394416424.dkr.ecr.us-east-1.amazonaws.com/aws-for-fluent-bit:2.0.0",
        "Version": 1,
        "LastModifiedDate": 1539908129.759,
        "ARN": "arn:aws:ssm:us-west-2::parameter/aws/service/aws-for-fluent-bit/2.0.0"
    }
}
```

#### Using SSM Parameters in CloudFormation Templates

You can use these SSM Parameters as parameters in your CloudFormation templates.

```
Parameters:
  FireLensImage:
    Description: Fluent Bit image for the FireLens Container
    Type: AWS::SSM::Parameter::Value<String>
    Default: /aws/service/aws-for-fluent-bit/latest
```

#### Using image tags

You should lock your deployments to a specific version tag. We guarantee that these tags will be immutable- once they are released the will not change.

##### Amazon ECR Public Gallery

[aws-for-fluent-bit](https://gallery.ecr.aws/aws-observability/aws-for-fluent-bit)

Our images are available in Amazon ECR Public Gallery. We recommend our customers to download images from this public repo. You can get images with different tags by following command:

```
docker pull public.ecr.aws/aws-observability/aws-for-fluent-bit:<tag>
```

For example, you can pull the image with latest version by:

```
docker pull public.ecr.aws/aws-observability/aws-for-fluent-bit:latest
```

If you see errors for image pull limits, try log into public ECR with your AWS credentials:

```
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws
```

You can check the [Amazon ECR Public official doc](https://docs.aws.amazon.com/AmazonECR/latest/public/get-set-up-for-amazon-ecr.html) for more details.

##### Docker Hub

[amazon/aws-for-fluent-bit](https://hub.docker.com/r/amazon/aws-for-fluent-bit/tags)

##### Amazon ECR

We also provide images in Amazon ECR for high availability. These images are available in almost every AWS region, included AWS Gov Cloud.

The official way to find the ECR image URIs for your region is to use the SSM Parameters. In your region, run the following command:

```
aws ssm get-parameters-by-path --path /aws/service/aws-for-fluent-bit/
```

### Plugins

We currently bundle the following projects in this image:
* [amazon-kinesis-firehose-for-fluent-bit](https://github.com/aws/amazon-kinesis-firehose-for-fluent-bit)
* [amazon-cloudwatch-logs-for-fluent-bit](https://github.com/aws/amazon-cloudwatch-logs-for-fluent-bit)
* [amazon-kinesis-streams-for-fluent-bit](https://github.com/aws/amazon-kinesis-streams-for-fluent-bit)

### Using the AWS Plugins outside of a container

You can use the AWS Fluent Bit plugins with [td-agent-bit](https://docs.fluentbit.io/manual/installation/supported-platforms).

We provide a [tutorial](examples/fluent-bit/systems-manager-ec2/) on using SSM to configure instances with td-agent-bit and the plugins.

### Development

#### Releasing

Use `make release` to build the image.
To run the integration tests, run `make integ-dev`. The `make integ-dev` command will run the integration tests for all of our plugins-
kinesis streams, kinesis firehose, and cloudwatch.

To run integration tests separately, execute `make integ-cloudwatch` or `make integ-kinesis` or `make integ-firehose`.

[Documentation on GitHub steps for releases](Release_Process.md).

#### Developing Features in the AWS Plugins

You can build a version of the image with code in your GitHub fork. To do so, you must need to set the following environment variables.
Otherwise, you will see an error message like the following one:
`fatal: repository '/kinesis-streams' or '/kinesis-firehose' or '/cloudwatch' does not exist.`

Set the following environment variables for CloudWatch:

```
export CLOUDWATCH_PLUGIN_CLONE_URL="Your GitHub fork clone URL"
export CLOUDWATCH_PLUGIN_BRANCH="Your branch on your fork"
```

Or for Kinesis Streams:
```
export KINESIS_PLUGIN_CLONE_URL="Your GitHub fork clone URL"
export KINESIS_PLUGIN_BRANCH="Your branch on your fork"
```

Or for Kinesis Firehose:
```
export FIREHOSE_PLUGIN_CLONE_URL="Your GitHub fork clone URL"
export FIREHOSE_PLUGIN_BRANCH="Your branch on your fork"
```

Then run `make cloudwatch-dev` or `make kinesis-dev` or `make firehose-dev` to build the image with your changes.

To run the integration tests on your code, execute `make integ-cloudwatch-dev` or `make integ-kinesis-dev` or `make integ-firehose-dev`.

## Fluent Bit Examples
Check out Fluent Bit examples from our [amazon-ecs-firelens-examples](https://github.com/aws-samples/amazon-ecs-firelens-examples#fluent-bit-examples) repo.

## License

This project is licensed under the Apache-2.0 License.
