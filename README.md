# EC2 ImageBuilder pipeline for container images

## Introduction

Building up-to-date container images is a key function needed to run containerized infrastructure. Although existing tools enable the building of individual container images, customers need to run it every time either manually or with homegrown automation to produce new images with the latest software updates, manually test functionality, and validate security posture of images with their compliance teams.


On December 17, 2020, [AWS announced that customers of EC2 Image Builder can now build and test container images compliant with the Open Container Initiative (OCI) specification](https://aws.amazon.com/about-aws/whats-new/2020/12/ec2-image-builder-supports-container-images/). As a result, EC2 Image Builder can be used to automate the building of both – Virtual Machine and container images with similar workflows. Similar to Image Builder’s workflows to build VM images, when software updates become available, Image Builder can automatically produce new up-to-date container images and publish them to specified Amazon Elastic Container Registry (ECR) repositories after running stipulated tests.


While customers can currently create an automated container image build pipeline in the AWS Management Console, the AWS CloudFormation documentation still does not provide all of the required Imagebuilder resources needed to generate a container image pipeline through AWS CloudFormation infrastructure as code (IaC). The below solution seeks to overcome this obstacle. 
___

## About this CloudFormation template

This sample template creates AWS CloudFormation resources for an EC2 ImageBuilder pipeline that builds an Amazon Linux 2 container image with Docker and publishes the image to the specified Amazon Elastic Container Registry (ECR) repository. The pipeline, by default, is scheduled to run a build at 9:00AM Coordinated Universal Time (UTC) on the first day of every month.


If you do not have a default VPC, or want to use a custom VPC, you will need to specify a subnet ID and one or more security group IDs in the VPC as parameters when you create a stack based on this template.
___

## What resources does this CloudFormation stack deploy?

First, the stack creates an [`AWS::S3::Bucket`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html), where logs are stored.
> **NOTE:**
> When deleting the stack, make sure to empty the bucket first. Otherwise, you will experience a status of "DELETE_FAILED" in CloudFormation.
> If you want to delete the stack but keep the bucket, set the [DelectionPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) to Retain (uncomment in template).


Additionally, an [`AWS::ECR::Repository`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecr-repository.html) resource is created, where EC2 Image builder can push the container image to after creation.
> **NOTE:**
> When deleting the stack, make sure to empty the repository first. Otherwise, you will experience a status of "DELETE_FAILED" in CloudFormation.

> Since no repository policy is defined in this example, the AWS managed policy "EC2InstanceProfileForImageBuilderECRContainerBuilds" is applied to the instance role / instance profile to grant appropriate permissions to upload ECR images.


By default, AWS Services do not have permission to perform actions on your instances. 
The [`AWS::IAM::Role`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html) grants AWS Systems Manager (SSM) and EC2 Image Builder the necessary permissions to build a container image / upload ECR images.


An [`AWS::IAM::Policy`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-policy.html) is generated to allow the EC2 instance to write logs to the S3 bucket (via instance role / instance profile).


To pass the instance role to an EC2 instance, we need an [`AWS::IAM::InstanceProfile`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-instanceprofile.html). This profile will be used during the container image build process.


The [`AWS::ImageBuilder::InfrastructureConfiguration`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-imagebuilder-infrastructureconfiguration.html) specifies the infrastructure within which to build and test your image.
The instance profile is specified as a parameter, telling EC2 Image Builder to use the specified profile with the EC2 instance during the build.
> **NOTE:**
> If you would like to keep the instance running after a failed build, set TerminateInstanceOnFailure to false. When providing parameters for this stack, define one or more instance types to use when building the instance.
EC2 Image Builder will select a type based on availability.


The stack creates a sample "Hello World" [`AWS::ImageBuilder::Component`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-imagebuilder-component.html) that demonstrates defining the build, validation, and test phases for an image build lifecycle. 
The component includes a validation step which will run after the install but before the image capture. Also included, is a test step which runs after the image is captured (EC2 Image Builder launches a new instance from the image and runs the test phase).


[`AWS::ImageBuilder::ContainerRecipe`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-imagebuilder-containerrecipe.html) defines how images are configured, tested, and assessed. This resource references the Parent Image tag specified in the stack parameters.
This resource requires the user to define a Dockerfile inline or specify an S3 URI for the Dockerfile.***
> **NOTE:**
> ***Dockerfiles are text documents that are used to build Docker containers and ensure that they contain all of the elements required by the application running inside. The template data consists of contextual variables where Image Builder places build information or scripts, based on your container image recipe.

> When using a custom source image (e.g. from DockerHub), make sure to specify the appropriate operating system platfrom via the "PlatformOverride" property.


The resource [`AWS::ImageBuilder::DistributionConfiguration`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-imagebuilder-distributionconfiguration.html) allows you to specify the name and description of your output container image and settings for encryption, licensing, and sharing to a target ECR repository in a specific region.
> **NOTE:**
> Users have the option to tag their container images that will populate in ECR upon successful image creation by using the "ContainerTags" property. By default, if no container tags are specified, the "Image tags" value in ECR will reflect the image build version (e.g. 1.0.0-1), which can be found by navigating to EC2 ImageBuilder in the AWS Management Console.


[`AWS::ImageBuilder::ImagePipeline`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-imagebuilder-imagepipeline.html) is the automation configuration for building secure OS images on AWS. The pipeline is associated with the ContainerRecipe and can also be associated with an infrastructure configuration and distribution configuration. You can also use a schedule to configure how often and when a pipeline will automatically create a new image.
> **NOTE:**
> In this example, a schedule expression is defined that triggers a build at 9:00AM Coordinated Universal Time (UTC) on the first day of every month. This can be customized by modifying the cron expression in the template.


Lastly, the [`AWS::ImageBuilder::Image`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-imagebuilder-image.html) is used to define the image build version. This resource will show a status of "CREATE_COMPLETE" in CloudFormation once your image is done building.
___

## Walkthrough

It should take approximately 15-20 minutes for the CloudFormation stack build to complete.

This solution can be deployed using both the AWS Management Console or the Command Line Interface (CLI). 


### AWS Management Console:
1. Login to your AWS account using the AWS Management Console and navigate to CloudFormation.
2. Create a new stack and upload the `ec2_imagebuilder_pipeline_for_container_images.yml` template file.
3. You will see a checkbox informing you that the stack creates IAM resources. Read and check the box.
4. Wait for the stack to build. Feel free to monitor the status of your container image build by navigating to Systems Manager (SSM) or EC2 Image Builder. Note that The `AWS::ImageBuilder::Image` resource will show a status of "CREATE_IN_PROGRESS" while the image is being created, and will later show "CREATE_COMPLETE" when the container image build is complete.

### CLI:
1. Ensure that your YAML template and JSON parameters file are located within your current directory.
2. Run the following command from your terminal:
```
aws cloudformation create-stack \
--stack-name sample-ec2-ib-pipeline-container-images \
--template-body file://ec2_imagebuilder_pipeline_for_container_images.yml \
--parameters file://ec2_imagebuilder_pipeline_for_container_images.json \
--capabilities CAPABILITY_NAMED_IAM \
--region us-east-1
```
> **NOTE:**
> A parameters file was created as part of this solution to simplify this process.
___

## Troubleshooting

While the stack is building, you will see an EC2 instance running. This is either the build or test instance. AWS Systems Manager (SSM) Automation will also run simultaneously. You can observe this automation to see the steps EC2 Image Builder takes to build your container image.


If the stack fails, check the CloudFormation "Events" tab. These events include a description of any failed resources. For additional troubleshooting associated with the "BUILD" or "TEST" phases, you can also navigate to the S3 bucket resource that was created as part of the the stack.
___

## Cleanup your stack

To delete AWS resources created by this stack:

1. Delete the contents of the S3 bucket created by the stack (if the bucket is not empty, the stack deletion will fail).To keep the bucket, uncomment the Retain deletion policy for the CloudFormation bucket resource.
2. Delete the container image within your ECR repository created by the stack (if the repo is not empty, the stack deletion will fail). Unfortunately, there is no Retain deletion policy for the CloudFormation ECR repository resource at this time.
3. Delete the stack in the CloudFormation console, or by using the CLI/SDK.


