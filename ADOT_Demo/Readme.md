# Lab Overview 

During this lab, we'll be implementing a robust observability solution for our cluster running on Amazon Elastic Kubernetes Service (EKS) using the AWS Distro for OpenTelemetry (ADOT) operator. With the ADOT collector, we'll export our metrics to AWS Managed Prometheus, allowing us to easily analyze and visualize our data in AWS Managed Grafana. In addition, we'll configure our ADOT collector to export traces to AWS X-Ray, enabling us to gain insights into the performance and behavior of our system.

   ## Pre-requisites

1. Create cluster using command
    
    ```html
    eksctl create cluster --name <cluster-name> --node-type <node-type> --nodes <node-count> --region <region-name>
    ```
    
2. Install kubectl CertManager
    
    ```html
    kubectl apply -f \
    https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
    ```
    
    ```html
    kubectl get pod -w -n cert-manager
    ```
    
3. Go to EKS Console and in Add ons add AWS Distro for Open Telemetry add on:

![image](https://user-images.githubusercontent.com/79714302/230729016-473456ca-cfc4-436e-8b9f-e720c2694454.png)


4. Do `kubectl get pods -A` to check if the ADOT operator pod is up and running or not.
5. Create Amazon Managed Prometheus and Amazon Managed Grafana workspaces(preferably in the same region where cluster is running)
6. Setup a Service Account

    a. Follow the below steps to setup IAM OIDC provider for your cluster
    
    ![image](https://user-images.githubusercontent.com/79714302/230728938-ae6f153b-2236-4437-9cc8-fa28f15fcaf7.png)

    
    b. Copy the Resource arn from the Summary page of your AWS Managed Prometheus(AMP) workspace and create this policy in AWS CLI
    
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "aps:RemoteWrite"
                ],
                "Resource": [
                    "arn:aws:aps:<region>:<account_id>:workspace/<workspace ID>"
                ]
            }
        ]
    }
    ```
    
    c. Replace *`my-service-account`* with the name of the Kubernetes service account that you want `eksctl` to create and associate with an IAM role. Replace *`default`* with the namespace that you want `eksctl` to create the service account in. Replace *`my-cluster`* with the name of your cluster. Replace *`my-role`* with the name of the role that you want to associate the service account to. If it doesn't already exist, `eksctl` creates it for you. Replace *`111122223333`* with your account ID , *`my-policy`* with the name of an existing policy and replace *`my-region`* with the region of your EKS clsuter.
    
    ```html
    eksctl create iamserviceaccount --name <my-service-account> --namespace <default> --cluster <my-cluster> --region <my-region> --role-name <"my-role"> \
        --attach-policy-arn <arn:aws:iam::111122223333:policy/my-policy> --approve
    ```
    
    d. Run this command to see if you setup your Service Account properly or not.Replace `<my-service-account>` with the name of your Service account and mention the namespace in which that service account was created in place of `<default>`
    
    ```html
    kubectl describe serviceaccount <my-service-account> -n <default>
    ```
    

## Setting up the Collector

1. Download the config file for the amp collector
    
    ```html
    curl -O https://raw.githubusercontent.com/aws-observability/aws-otel-community/master/sample-configs/operator/collector-config-amp.yaml
    ```
    
2. In `collector-config-amp.yaml`, replace the following with your own values:
    - *`mode: deployment`*
    - *`serviceAccount: adot-collector`*(replace this with the name of the service Account we created)
    - *`endpoint: "<YOUR_REMOTE_WRITE_ENDPOINT>"`*
    - *`region: "<YOUR_AWS_REGION>"`*
    - *`name: adot-collector`* 
3. Apply the yaml file with this command
    
    ```html
    kubectl apply -f collector-config-amp.yaml
    ```
    
4. Run below command and verify there is a pod in default namespace running as amp-collector

```html
kubectl get pods -A
```

## Integrating AMP with AWS Managed Grafana

1. After the config file has been deployed, we will login to AWS Grafana worspace and add a new Data Source
    
    ![image](https://user-images.githubusercontent.com/79714302/230540493-9b1a7437-ba52-43bd-9b93-695af8cc01f3.png)
    
2. Go and project data onto a Dashboard of your choice
    
    ![image](https://user-images.githubusercontent.com/79714302/230540527-63ecc6f2-ef8d-496e-8da4-039e67fc7100.png)
    
    ![image](https://user-images.githubusercontent.com/79714302/230540562-7ff592bb-2cd9-4572-bcc1-846b75ae2f1e.png)
    
    ![image](https://user-images.githubusercontent.com/79714302/230540587-a2abb0ae-4aca-44ee-85bc-b13b85c50078.png)
    

# Additional Steps to Integrate AWS X-ray using ADOT

## Pre-requisites 2

1. Create an IAM Policy from the AWS CLI with the following permissions
    
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "CloudWatchLogsAccess",
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Resource": "arn:aws:logs:*:*:*"
            },
            {
                "Sid": "XRayAccess",
                "Effect": "Allow",
                "Action": [
                    "xray:PutTraceSegments",
                    "xray:PutTelemetryRecords"
                ],
                "Resource": "*"
            },
            {
                "Sid": "Cloudwatch",
                "Action": [
                    "cloudwatch:PutMetricData"
                ],
                "Resource": "*",
                "Effect": "Allow"
            },
            {
                "Sid": "ELB",
                "Action": [
                    "ec2:DescribeAccountAttributes",
                    "ec2:DescribeAddresses",
                    "ec2:DescribeInternetGateways"
                ],
                "Resource": "*",
                "Effect": "Allow"
            }
        ]
    }
    ```
    
2. Attach this policy to previously created AWS IAM role.This way the access to AWS X-Ray is given to the Service Account as well which is going to be used by the pods to export their traces.
3. Lets setup a Sample Application and Traffic Generator in order to create some traces that will reflect in our X-Ray Console
  
    1. Lets deploy the traffic Generator first which will be creating traffic on the port 4567
        
        ```html
        curl -O https://raw.githubusercontent.com/aws-observability/aws-otel-community/master/sample-configs/traffic-generator.yaml
        ```
        
        ```html
        kubectl apply -f traffic-generator.yaml
        ```
        
    2. Lets deploy the Sample Application now
        a. Run the below command: 
        
        ```html
        curl -O https://raw.githubusercontent.com/aws-observability/aws-otel-community/master/sample-configs/sample-app.yaml
        ```
        
        b. Update the value for “<YOUR_AWS_REGION>” for the region in which your cluster is deployed
        
        c. Replace the value for “OTEL_EXPORTER_OTLP_ENDPOINT” with “[http://my-collector-xray-collector:4317](http://my-collector-xray-collector:4317/)” or replace my-collector-xray with whatever the name of your xray-collector is going to be and 4317 is the port where the app with forward the traces to xray.
        
        d. Deploy the sample app
        
        ```html
        kubectl apply -f sample-app.yaml
        ```
        

## Deploying the xray collector:


1. Download the xray collector configuration file
    
    ```html
    curl -O https://raw.githubusercontent.com/aws-observability/aws-otel-community/master/sample-configs/operator/collector-config-xray.yaml
    ```
    
2. In `collector-config-xray.yaml`, replace the following with your own values:
    - *`mode: deployment`*
    - *`serviceAccount: adot-collector`*(provide the value of the Service Account we created)
    - *`region: "<YOUR_AWS_REGION>"`*
3. Deploy the collector
    
    ```html
    kubectl apply -f collector-config-xray.yaml
    ```
    

### Finally go to AWS X-Ray console and click on the traces tab

![image](https://user-images.githubusercontent.com/79714302/230540837-136a7d6a-905b-43eb-aebc-aefff32586f0.png)

![image](https://user-images.githubusercontent.com/79714302/230540870-ab4688da-cc1e-41b5-b312-377d636a54aa.png)

## Cleanup
   
   Run the following command and replace cluster name and region with your own details to delete your EKS Cluster
   
   ```html
   eksctl delete cluster --name <cluster-name> --region <region-name>
   ```
   
