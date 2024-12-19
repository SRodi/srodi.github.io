---
title: "How to test Terraform modules: a simple tutorial using IBM Cloud provider"
date: 2023-09-19 09:00:00 +0100
categories: [devops, iac]
tags: [terraform, ibmcloud, terratest, go, testing]     ## TAG names should always be lowercase
image:
  path: /intro-image.webp
  alt: "A robust approach to test Terraform modules"
---

## Introduction
As part of a production grade Terraform deployment, testing plays an integral part. This tutorial will demonstrate how to write and effectively test a Terraform module before deploying to a production environment.

## Why testing Infrastructure as Code?
Testing Infrastructure as Code (IaC) is as essential as testing any other software code. Resource outputs of Terraform modules are normally used and consumed by other Terraform resources part of a cloud environment stack. If the outputs are not correct, we could break entire production systems. This not only occurs at resource creation, but also when updating the Terraform configuration for the same resources.

For a production grade Terraform deployment, IaC testing should be performed in automation with dedicated CI pipelines.

## Tutorial goals
In this tutorial we will perform the following:

* Create an IBM Cloud Object Storage (COS) instance and bucket via Terraform
* Create a resource key with Object Writer permissions to the bucket via Terraform
* Validate Terraform resources are correctly configured
* Test COS instance and bucket creation
* Test writer key can upload an object to the bucket
* Test writer key cannot delete an object stored in the bucket

## High level plan
We will create the infrastructure with Terraform in a modularised approach, for the purpose of the tutorial we create one single module. Then we will test the module by creating an example which instantiate the module implementation. Finally we will perform tests using Terratest, which is a Go testing library for terraform, and also IBM Cloud COS SDK for Go, to test with a real client and real resources.

## Prerequisites
The following are requirements for this tutorial, please make sure you have an IBM Cloud account with and apiKey available which has full permissions for COS service. In addition you need to have Terraform, Go and Git installed on your local machine.

* IBM Cloud account
* Terraform CLI
* Go
* Git

The versions I have currently installed on my environment are as follows:

```bash
❯ terraform version
Terraform v1.3.9
on darwin_amd64
❯ go version
go version go1.20.6 darwin/amd64
❯ git version
git version 2.39.2 (Apple Git-143)
```

## Tutorial
Lets deep dive into our tutorial. If you are too eager to get to the final solution, please feel free to check out the code at this [link Tutorial GitHub repo](https://github.com/SRodi/tf-bg-1)

### Part 1: Directory structure for cos module folder
To start we will create the initial directory structure for the module. For the purpose of this tutorial we will create only one module named cos.

```
tf-bg-1
└── modules
    └── cos
       ├── main.tf
       ├── outputs.tf
       └── providers.tf
```

### Part 2: Create a custom Terraform module with IBM Cloud resource definitions
In the _main.tf_ file we define all IBM Cloud resources for the creation of a COS instance, bucket and resource keys to interact with the bucket.

```terraform
data "ibm_resource_group" "resource_group_default" {
  name = "Default"
}

resource "ibm_resource_instance" "resource_instance_cos_test" {
  name              = "cos-instance-test"
  service           = "cloud-object-storage"
  plan              = "lite"
  location          = "global"
  resource_group_id = data.ibm_resource_group.resource_group_default.id
}

resource "ibm_cos_bucket" "bucket_test" {
  bucket_name          = "cos-bucket-tf-test"
  resource_instance_id = ibm_resource_instance.resource_instance_cos_test.id
  region_location      = "eu-gb"
  storage_class        = "standard"
}

resource "ibm_resource_key" "key_cos_object_writer" {
  name                 = "cos-key-object-writer-test"
  role                 = "Object Writer"
  resource_instance_id = ibm_resource_instance.resource_instance_cos_test.id
}
```

In _providers.tf_ we will define the configuration for the IBM Cloud Terraform provider

```terraform
terraform {
  required_version = ">=1.3.0, <1.6"
  required_providers {
    ibm = {
      source  = "IBM-Cloud/ibm"
      version = "1.57.0"
    }
  }
}
```

In _outputs.tf_ we define the output for COS module, outputs are important to enable testing and allow integrating with other modules part of the Terraform stack. In this tutorial we have only one module, but we will create an example with an instance of this module which will also consume the output defined in the module implementation below.

```terraform
output "key_object_writer" {
  value     = ibm_resource_key.key_cos_object_writer.credentials["apikey"]
  sensitive = true
}

output "service_instance_id" {
  value = ibm_resource_instance.resource_instance_cos_test.guid
}

output "bucket_name" {
  value = ibm_cos_bucket.bucket_test.bucket_name
}
```

### Part 3: Directory structure for example folder
The example directory contain a subdirectory cos with the implementation of the example. The implementation consists in the instantiation of COS module.

```
tf-bg-1
├── examples
│   └── cos
│       ├── main.tf
│       ├── outputs.tf
│       └── providers.tf
└── modules
```

The file _main.tf_ contains the cos example with the instantiation of COS module

```terraform
module "cos" {
  source = "../../modules/cos"
}
```

The file _output.tf_ contains the outputs of COS example

```terraform
output "key_object_writer" {
  value     = module.cos.key_object_writer
  sensitive = true
}

output "service_instance_id" {
  value = module.cos.service_instance_id
}

output "bucket_name" {
  value = module.cos.bucket_name
}
```

The file _providers.tf_ contains the providers configuration for COS example

```terraform
terraform {
  required_version = ">=1.3.0, <1.6"
  required_providers {
    ibm = {
      source  = "IBM-Cloud/ibm"
      version = "1.57.0"
    }
  }
}
```

At this point we already have a working example. If we run terraform init, terraform plan and terraform apply within the examples/cos directory we can already interact with the IBM Cloud Terraform provider and create the resources defined in modules/cos.

### Part 4: Directory structure for tests folder
_tests_ folder contains _cos_test.go_ with the test implementation for cos example, _go.mod_ file defines go module’s module path, _go.sum_ containing the expected cryptographic hashes of the content of specific module version.

```
tf-bg-1
├── tests
│   ├── cos_test.go
│   ├── go.mod
│   └── go.sum
├── tests
└── modules
```

### Part 5: Testing COS module example with Terratest
In this part we test the COS module with the Go testing library Terratest, which provides patterns and helper functions for testing infrastructure, with 1st-class support for Terraform.

The code below defines the Terraform options struct, which in this case contains only the Terraform directory that references a local path where our example implementation is. In addition the code runs a Terraform init, plan and apply. Finally the Terraform destroy operation will run at the end of every other instruction, to clean up at the end of the test, this is achieved using the defer keyword, to ensure the operation is run before the function returns, despite eventual failure of other instructions.

```go
func TestInfraCOSExample(t *testing.T) {
	opts := &terraform.Options{
		TerraformDir: "../examples/cos",
	}
	// clean up at the end of the test
	defer terraform.Destroy(t, opts)
	terraform.Init(t, opts)
	terraform.Apply(t, opts)
}
```

### Part 6: Configure the environment and run the test
In order to run the test you will have to firstly export the environment variables required by the IBM Cloud Terraform provider for authentication.

```bash
export IC_API_KEY=yourSuperSecretApiKey
export IC_REGION=eu-gb
```

Then you can run the test as follows:

```bash
cd tests
go test -run TestInfraCOSExample
```

### Part 7: Get Terraform module outputs during test execution
Before proceeding with the actual test we need to make sure the cos example outputs are available in our go test. This can be achieved by calling the _terraform.OutputRequired()_ function.

```go
func TestInfraCOSExample(t *testing.T) {
	opts := &terraform.Options{
		TerraformDir: "../examples/cos",
	}
	// clean up at the end of the test
	defer terraform.Destroy(t, opts)
	terraform.Init(t, opts)
	terraform.Apply(t, opts)

	serviceInstanceId := terraform.OutputRequired(t, opts, "service_instance_id")
	bucketName := terraform.OutputRequired(t, opts, "bucket_name")

	fmt.Println(serviceInstanceId,bucketName)   
}
```

### Part 8: Configure the IBM Cloud COS SDK client
We can now leverage the IBM Cloud COS SDK to interact with the COS instance and bucket to fully test the resources created with Terraform and the output provided are usable without any manipulation. This function creates an S3 client by instantiating a session with an IBM Cloud COS instance.

```go
func createClient(apiKey, serviceInstanceID string) *s3.S3 {
	conf := aws.NewConfig().
		WithRegion("us-standard").
		WithEndpoint(serviceEndpoint).
		WithCredentials(ibmiam.NewStaticCredentials(aws.NewConfig(), authEndpoint, apiKey, serviceInstanceID)).
		WithS3ForcePathStyle(true)
	clientSession := session.Must(session.NewSession())
	client := s3.New(clientSession, conf)
	return client
}
```

### Part 9: Test writer key can upload an object to the COS bucket
The function below instantiates a client to communicate with IBM Cloud COS, it uploads an object and verifies that the object was uploaded correctly.

```go
func testUploadObject(t *testing.T, apiKey, serviceInstanceID, bucketName string) {
	client := createClient(apiKey, serviceInstanceID)
	content := bytes.NewReader([]byte(testObjectContent))
	result, err := client.PutObject(&s3.PutObjectInput{
		Bucket: aws.String(bucketName),
		Key:    aws.String(testObjectKey),
		Body:   content,
	})
	if err != nil {
		log.Panic(err)
		t.Fail()
	}
	// test ETag exists, meaning the object is uploaded
	assert.NotEmpty(t, result.ETag)
}
```

### Part 10: Test writer key can upload an object to the COS bucket
This function uses the COS client to test an object delete. Given that the apikey has _Object Writer_ permissions the object delete attempt should fail.

```go
func testDeleteObject(t *testing.T, apiKey, serviceInstanceID, bucketName string) {
	client := createClient(apiKey, serviceInstanceID)
	_, err := client.DeleteObject(&s3.DeleteObjectInput{
		Bucket: aws.String(bucketName),
		Key:    aws.String(testObjectKey),
	})
	if err != nil {
		if awsError, ok := err.(awserr.Error); ok {
			// if there is an error we expect a specific failure
			// in this case AccessDenied as the service key
			// does not have permission to delete the object
			assert.Equal(t, "AccessDenied", awsError.Code())
		} else {
			log.Panic(err)
			t.Fail()
		}
	}
}
```

### Part 11: Putting it all together
Here is the final code for this simple test.

```go
package test

import (
	"bytes"
	"log"
	"testing"
	"github.com/IBM/ibm-cos-sdk-go/aws"
	"github.com/IBM/ibm-cos-sdk-go/aws/credentials/ibmiam"
	"github.com/IBM/ibm-cos-sdk-go/aws/session"
	"github.com/IBM/ibm-cos-sdk-go/service/s3"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

const (
	authEndpoint      = "https://iam.cloud.ibm.com/identity/token"
	serviceEndpoint   = "https://s3.eu-gb.cloud-object-storage.appdomain.cloud"
	testObjectKey     = "testKey1"
	testObjectContent = "some testing random text"
)


func createClient(apiKey, serviceInstanceID string) *s3.S3 {
	conf := aws.NewConfig().
		WithRegion("us-standard").
		WithEndpoint(serviceEndpoint).
		WithCredentials(ibmiam.NewStaticCredentials(aws.NewConfig(), authEndpoint, apiKey, serviceInstanceID)).
		WithS3ForcePathStyle(true)
	clientSession := session.Must(session.NewSession())
	client := s3.New(clientSession, conf)
	return client
}

func testUploadObject(t *testing.T, apiKey, serviceInstanceID, bucketName string) {
	client := createClient(apiKey, serviceInstanceID)
	content := bytes.NewReader([]byte(testObjectContent))
	result, err := client.PutObject(&s3.PutObjectInput{
		Bucket: aws.String(bucketName),
		Key:    aws.String(testObjectKey),
		Body:   content,
	})
	if err != nil {
		log.Panic(err)
		t.Fail()
	}
	// test ETag exist, meaning the object is uploaded
	assert.NotEmpty(t, result.ETag)
}

func testDeleteObject(t *testing.T, apiKey, serviceInstanceID, bucketName string) {
	client := createClient(apiKey, serviceInstanceID)
	_, err := client.DeleteObject(&s3.DeleteObjectInput{
		Bucket: aws.String(bucketName),
		Key:    aws.String(testObjectKey),
	})
	if err != nil {
		if awsError, ok := err.(awserr.Error); ok {
			// if there is an error we expect a specific failure
			// in this case AccessDenied as the service key
			// does not have permission to delete the object
			assert.Equal(t, "AccessDenied", awsError.Code())
		} else {
			log.Panic(err)
			t.Fail()
		}
	}
}

func TestInfraCOSExample(t *testing.T) {
	opts := &terraform.Options{
		TerraformDir: "../examples/cos",
	}
	// clean up at the end of the test
	defer terraform.Destroy(t, opts)
	terraform.Init(t, opts)
	terraform.Apply(t, opts)

	keyObjectWriter := terraform.OutputRequired(t, opts, "key_object_writer")
	serviceInstanceId := terraform.OutputRequired(t, opts, "service_instance_id")
	bucketName := terraform.OutputRequired(t, opts, "bucket_name")

	// test keyObjectWriter, if this test pass
	// we are confident Terraform output is valid and we have defined
	// the correct Write permission for this service key
	testUploadObject(t, keyObjectWriter, serviceInstanceId, bucketName)

	// test delete function with both keyObjectWriter and keyReader
	// expect failure as we did not set Delete permission to keys
	testDeleteObject(t, keyObjectWriter, serviceInstanceId, bucketName)

	// test object can be deleted with provisioning apiKey
	// which has full COS permissions. expect success
	testDeleteObject(t, os.Getenv("IC_API_KEY"), serviceInstanceId, bucketName)
}
```

## Conclusion
In this tutorial we have first created a small Terraform project using a modularised approach, then we created an example which instantiates the module, and finally we tested the example using Terratest and IBM Cloud COS Go SDK.

Testing sensitive credentials outputs is particularly important as these should be stored as secrets and securely consumed by applications running in production. This process should be hands-off, and the actual secret value should never be exposed or manually handled. The implementation of an automated test provides the necessary confidence to push changes to production while knowing the given credential exists, it is in the right format, and it is configured with the correct permissions.

Note: I have purposely omitted defining the type of test as “unit test” or “integration test” since this is neither nor both. Some people define this as a unit test, where the unit is represented by the Terraform module but in reality this is more like an integration test since we create actual resources by interacting with a real Terraform provider. In addition, some could claim this is closer to an “end to end” test since we also use a COS client, but this is not necessarily the client implementation we have in production right? Personally, I think it is not important to define the name and type of the test, as long as we are actually fully testing the module prior to deploying in a production environment!

Thanks for following along, and I hope you enjoyed this content!

## References
* IBM Cloud docs: [cloud-object-storage-using-go](https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-using-go)
* Tutorial code with more tests: [GitHub repository](https://github.com/SRodi/tf-bg-1)
* Terraform providers registry: [IBM-Cloud](https://registry.terraform.io/providers/IBM-Cloud/ibm/latest)
* Terratest testing library: [Docs](https://terratest.gruntwork.io/docs/)

## Note
This article was also published on Medium, link [here](https://medium.com/@simone.rodigari/how-to-test-terraform-modules-542ac88f90b3)
