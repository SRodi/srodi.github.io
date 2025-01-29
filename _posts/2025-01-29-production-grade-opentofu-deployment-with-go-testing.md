---
title: "Production grade IaC deployments using go testing framework"
date: 2025-01-29 07:00:00 +0100
categories: [iac, go]
tags: [opentofu, terraform, go, kubernetes, azure, aks] ## always lowercase !!
mermaid: true
image:
  path: /iac-go-test.webp
  alt: "A robust approach to validating IaC deployments."
---

## Introduction
When deploying Infrastructure as Code (IaC), provisioning is just the first step—ensuring that the infrastructure functions as expected is equally critical. A successful deployment doesn't necessarily mean a fully operational system. Testing the outputs and interacting with the deployed resources is key to verifying real-world usability.

In this blog post, I’ll demonstrate how I use the Terratest Go library to automate the validation of an Azure Kubernetes Service (AKS) cluster deployed with OpenTofu. This approach has been invaluable across small, medium, and large-scale projects, saving me countless hours debugging and fixing infrastructure issues. By integrating testing directly into the deployment pipeline, I ensure that my infrastructure is not only successfully provisioned but also production-ready.

The code repository referenced in this blog post is accessible [here](https://github.com/SRodi/azure-aks).

## Why Test IaC?
Without automated validation, infrastructure deployments can lead to hidden issues such as:

* Misconfigured networking preventing application communication

* Incorrect permissions breaking access to critical resources

* Missing or improperly set environment variables

* Services failing to start despite successful provisioning

Automated testing with Terratest allows us to catch these issues early by interacting with the deployed infrastructure and verifying its functionality.


## Example: Validating an AKS Cluster Deployment
In the following example, I use Terratest to deploy an AKS cluster with OpenTofu and verify its functionality. The test:

1. Deploys the AKS cluster using OpenTofu.

2. Retrieves necessary outputs such as certificates and host information.

3. Uses these outputs to create a Kubernetes client.

4. Interacts with the cluster to ensure it is functional by listing namespaces.

This is an example for the main test function.

```go
package test

import (
	"context"
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func TestAKSCluster(t *testing.T) {
	t.Parallel()

	// Define the OpenTofu options
	terraformOptions := &terraform.Options{
		// The path to where your OpenTofu code is located
		TerraformDir: "../examples/aks",

		// Variables to pass to our OpenTofu code using -var options
		Vars: map[string]interface{}{
			"resource_group_name": "test-rg",
			"location":            "uksouth",
			"prefix":              "test",
			"labels": map[string]string{
				"env": "test",
			},
		},
	}

	// Clean up resources with "tofu destroy" at the end of the test
	defer terraform.Destroy(t, terraformOptions)
	// Run "tofu init" and "tofu apply". Fail the test if there are any errors.
	terraform.InitAndApply(t, terraformOptions)

	// Fetch the outputs
	caCert := fetchSensitiveOutput(t, terraformOptions, "ca_certificate")
	clientKey := fetchSensitiveOutput(t, terraformOptions, "client_key")
	clientCert := fetchSensitiveOutput(t, terraformOptions, "client_certificate")
	host := fetchSensitiveOutput(t, terraformOptions, "host")

	// Decode the base64 encoded strings
	caCertDecoded := decodeBase64(t, caCert)
	clientKeyDecoded := decodeBase64(t, clientKey)
	clientCertDecoded := decodeBase64(t, clientCert)

	// Create a new REST config using the outputs
	restConfig := newRESTConfig(caCertDecoded, clientKeyDecoded, clientCertDecoded, host)

	// Create a new Kubernetes client using the REST config
	k8sClient, err := newK8sClient(restConfig)
	if err != nil {
		t.Fatalf("Failed to create Kubernetes client: %v", err)
	}

	// Use the Kubernetes client to interact with the cluster
	_, err = k8sClient.CoreV1().Namespaces().List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		t.Fatalf("Failed to list namespaces: %v", err)
	}
}
```

Here is an example of the supporting test functions:

```go
package test

import (
	"encoding/base64"
	"fmt"
	"testing"

	"github.com/gruntwork-io/terratest/modules/logger"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

// fetch the sensitive output from OpenTofu
func fetchSensitiveOutput(t *testing.T, options *terraform.Options, name string) string {
	defer func() {
		options.Logger = nil
	}()
	options.Logger = logger.Discard
	return terraform.Output(t, options, name)
}

// decode the base64 encoded string
func decodeBase64(t *testing.T, encoded string) string {
	decodedBytes, err := base64.StdEncoding.DecodeString(encoded)
	if err != nil {
		t.Fatalf("Failed to decode base64 string: %v", err)
	}
	return string(decodedBytes)
}

// newK8sClient creates a new Kubernetes client using REST config
func newK8sClient(restConfig *rest.Config) (*kubernetes.Clientset, error) {
	clientset, err := kubernetes.NewForConfig(restConfig)
	if err != nil {
		return nil, fmt.Errorf("failed to create Kubernetes client: %v", err)
	}
	return clientset, nil
}

// creates a new REST config using the provided options
func newRESTConfig(caCert, clientKey, clientCert, host string) *rest.Config {
	return &rest.Config{
		Host: host,
		TLSClientConfig: rest.TLSClientConfig{
			CAData:   []byte(caCert),
			CertData: []byte(clientCert),
			KeyData:  []byte(clientKey),
		},
	}
}
```


### Breakdown of the Test
* Provisioning the AKS Cluster: OpenTofu is used to deploy the cluster.

* Retrieving Outputs: The test extracts sensitive outputs needed to authenticate with the cluster.

* Creating a Kubernetes Client: These outputs are used to establish a connection with the cluster.

* Validating Cluster Functionality: The test interacts with the Kubernetes API by listing namespaces, confirming that the cluster is accessible and functioning as expected.

## Conclusions
Automating IaC testing with Terratest ensures that infrastructure deployments are not only successful but also functional and production-ready. By interacting with the cluster after provisioning, we can verify that it meets our expectations before proceeding with application deployment.

This approach has saved me significant time debugging and fixing infrastructure issues. If you’re deploying Kubernetes clusters or any cloud infrastructure, consider integrating automated testing into your workflow—it will pay off in the long run!

The code repository referenced in this blog post is accessible [here](https://github.com/SRodi/azure-aks).
