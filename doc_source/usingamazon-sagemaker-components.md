# Using SageMaker Components<a name="usingamazon-sagemaker-components"></a>

In this tutorial, you run a pipeline using SageMaker Components for Kubeflow Pipelines to train a classification model using Kmeans with the MNIST dataset\. This workflow uses Kubeflow pipelines as the orchestrator and SageMaker as the backend to run the steps in the workflow\. For the full code for this and other pipeline examples, see the [Sample SageMaker Kubeflow Pipelines](https://github.com/kubeflow/pipelines/tree/master/samples/contrib/aws-samples)\. For information on the components used, see the [KubeFlow Pipelines GitHub repository](https://github.com/kubeflow/pipelines/tree/master/components/aws/sagemaker)\. 

**Topics**
+ [Setup](#setup)
+ [Running the Kubeflow Pipeline](#running-the-kubeflow-pipeline)

## Setup<a name="setup"></a>

To use Kubeflow Pipelines \(KFP\), you need an Amazon Elastic Kubernetes Service \(Amazon EKS\) cluster and a gateway node to interact with that cluster\. The following sections show the steps needed to set up these resources\. 

**Topics**
+ [Set up a gateway node](#set-up-a-gateway-node)
+ [Set up an Amazon EKS cluster](#set-up-anamazon-eks-cluster)
+ [Install Kubeflow Pipelines](#install-kubeflow-pipelines)
+ [Access the KFP UI](#access-the-kfp-ui)
+ [Create IAM Users/Roles for KFP pods and the SageMaker service](#create-iam-usersroles-for-kfp-pods-and-the-amazon-sagemaker-service)
+ [Add access to additional IAM users or roles](#add-access-to-additional-iam-users-or-roles)

### Set up a gateway node<a name="set-up-a-gateway-node"></a>

A gateway node is used to create an Amazon EKS cluster and access the Kubeflow Pipelines UI\. Use your local machine or an Amazon EC2 instance as your gateway node\. If you want to use a new Amazon EC2 instance, create one with the latest Ubuntu 18\.04 DLAMI version from the AWS console using the steps in [Launching and Configuring a DLAMI](https://docs.aws.amazon.com/dlami/latest/devguide/launch-config.html)\. 

Complete the following steps to set up your gateway node\. Depending on your environment, you may have certain requirements already configured\. 

1. If you don’t have an existing Amazon EKS cluster, create a user named `your_credentials` using the steps in [Creating an IAM User in Your AWS Account](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)\. If you have an existing Amazon EKS cluster, use the credentials of the IAM role or user that has access to it\. 

1. Add the following permissions to your user using the steps in [Changing Permissions for an IAM User:](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html#users_change_permissions-add-console) 
   + CloudWatchLogsFullAccess 
   + [AWSCloudFormationFullAccess](https://console.aws.amazon.com/iam/home?region=us-east-1#/policies/arn%3Aaws%3Aiam%3A%3Aaws%3Apolicy%2FAWSCloudFormationFullAccess) 
   + IAMFullAccess 
   + AmazonS3FullAccess 
   + AmazonEC2FullAccess 
   + AmazonEKSAdminPolicy \(Create this policy using the schema from [Amazon EKS Identity\-Based Policy Examples](https://docs.aws.amazon.com/eks/latest/userguide/security_iam_id-based-policy-examples.html)\) 

1. Install the following on your gateway node to access the Amazon EKS cluster and KFP UI\. 
   + [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)\. If you are using an IAM user, configure your [Access Key ID, Secret Access Key](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys) and preferred AWS Region by running: `aws configure` 
   + [aws\-iam\-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) version 0\.1\.31 and above 
   + `eksctl` version above 0\.15\. 
   + [https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl) \(The version needs to match your Kubernetes version within one minor version\) 

1. Install `boto3`\. 

   ```
   pip install boto3
   ```

### Set up an Amazon EKS cluster<a name="set-up-anamazon-eks-cluster"></a>

Run the following steps from the command line of your gateway node to set up an Amazon EKS cluster: 

1. If you do not have an existing Amazon EKS cluster, complete the following substeps\. If you already have an Amazon EKS cluster, skip this step\. 

   1. Run the following from your command line to create an Amazon EKS Cluster with version 1\.14 or above\. Replace `<your-cluster-name>` with any name for your cluster\. 

      ```
      eksctl create cluster --name <your-cluster-name> --region us-east-1 --auto-kubeconfig --timeout=50m --managed --nodes=1
      ```

   1. When cluster creation is complete, verify that you have access to the cluster using the following command\. 

      ```
      kubectl get nodes
      ```

1. Verify that the current `kubectl` context is the cluster you want to use with the following command\. The current context is marked with an asterisk \(\*\) in the output\. 

   ```
   kubectl config get-contexts
   
   CURRENT NAME     CLUSTER
   *   <username>@<clustername>.us-east-1.eksctl.io   <clustername>.us-east-1.eksctl.io
   ```

1. If the desired cluster is not configured as your current default, update the default with the following command\. 

   ```
   aws eks update-kubeconfig --name <clustername> --region us-east-1
   ```

### Install Kubeflow Pipelines<a name="install-kubeflow-pipelines"></a>

Run the following steps from the command line of your gateway node to install Kubeflow Pipelines on your cluster\. 

1. Install Kubeflow Pipelines on your cluster by following step 1 of [Deploying Kubeflow Pipelines documentation](https://www.kubeflow.org/docs/pipelines/installation/standalone-deployment/#deploying-kubeflow-pipelines)\. Your KFP version must be 0\.5\.0 or above\. 

1. Verify that the Kubeflow Pipelines service and other related resources are running\. 

   ```
   kubectl -n kubeflow get all | grep pipeline
   ```

   Your output should look like the following\. 

   ```
   pod/ml-pipeline-6b88c67994-kdtjv                      1/1     Running            0          2d
   pod/ml-pipeline-persistenceagent-64d74dfdbf-66stk     1/1     Running            0          2d
   pod/ml-pipeline-scheduledworkflow-65bdf46db7-5x9qj    1/1     Running            0          2d
   pod/ml-pipeline-ui-66cc4cffb6-cmsdb                   1/1     Running            0          2d
   pod/ml-pipeline-viewer-crd-6db65ccc4-wqlzj            1/1     Running            0          2d
   pod/ml-pipeline-visualizationserver-9c47576f4-bqmx4   1/1     Running            0          2d
   service/ml-pipeline                       ClusterIP   10.100.170.170   <none>        8888/TCP,8887/TCP   2d
   service/ml-pipeline-ui                    ClusterIP   10.100.38.71     <none>        80/TCP              2d
   service/ml-pipeline-visualizationserver   ClusterIP   10.100.61.47     <none>        8888/TCP            2d
   deployment.apps/ml-pipeline                       1/1     1            1           2d
   deployment.apps/ml-pipeline-persistenceagent      1/1     1            1           2d
   deployment.apps/ml-pipeline-scheduledworkflow     1/1     1            1           2d
   deployment.apps/ml-pipeline-ui                    1/1     1            1           2d
   deployment.apps/ml-pipeline-viewer-crd            1/1     1            1           2d
   deployment.apps/ml-pipeline-visualizationserver   1/1     1            1           2d
   replicaset.apps/ml-pipeline-6b88c67994                      1         1         1       2d
   replicaset.apps/ml-pipeline-persistenceagent-64d74dfdbf     1         1         1       2d
   replicaset.apps/ml-pipeline-scheduledworkflow-65bdf46db7    1         1         1       2d
   replicaset.apps/ml-pipeline-ui-66cc4cffb6                   1         1         1       2d
   replicaset.apps/ml-pipeline-viewer-crd-6db65ccc4            1         1         1       2d
   replicaset.apps/ml-pipeline-visualizationserver-9c47576f4   1         1         1       2d
   ```

### Access the KFP UI<a name="access-the-kfp-ui"></a>

The Kubeflow Pipelines UI is used for managing and tracking experiments, jobs, and runs on your cluster\. You can use port forwarding to access the Kubeflow Pipelines UI from your gateway node\. 

#### Set up port forwarding to the KFP UI service<a name="set-up-port-forwarding-to-the-kfp-ui-service"></a>

Run the following from the command line of your gateway node: 

1. Verify that the KFP UI service is running using the following command: 

   ```
   kubectl -n kubeflow get service ml-pipeline-ui
   
   NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
   ml-pipeline-ui   ClusterIP   10.100.38.71   <none>        80/TCP    2d22h
   ```

1. Run the following command to set up port forwarding to the KFP UI service\. This forwards the KFP UI to port 8080 on your gateway node and allows you to access the KFP UI from your browser\. 

   ```
   kubectl port-forward -n kubeflow service/ml-pipeline-ui 8080:80
   ```

   The port forward from your remote machine drops if there is no activity\. Run this command again if your dashboard is unable to get logs or updates\. If the commands return an error, ensure that there is no process already running on the port you are trying to use\. 

#### Access the KFP UI service<a name="set-up-port-forwarding-to-the-kfp-ui-service-access"></a>

Your method of accessing the KFP UI depends on your gateway node type\. 
+ Local machine as the gateway node 

  1. Access the dashboard in your browser as follows: 

     ```
     http://localhost:8080
     ```

  1. Choose **Pipelines** to access the pipelines UI\. 
+  Amazon EC2 instance as the gateway node 

  1. You need to set up an SSH tunnel on your Amazon EC2 instance to access the Kubeflow dashboard from your local machine’s browser\. 

     From a new terminal session in your local machine, run the following\. Replace `<public-DNS-of-gateway-node>` with the IP address of your instance found on the Amazon EC2 console\. You can also use the public DNS\. Replace `<path_to_key>` with the path to the pem key used to access the gateway node\. 

     ```
     public_DNS_address=<public-DNS-of-gateway-node>
     key=<path_to_key>
     
     on Ubuntu:
     ssh -i ${key} -L 9000:localhost:8080 ubuntu@${public_DNS_address}
     
     or on Amazon Linux:
     ssh -i ${key} -L 9000:localhost:8080 ec2-user@${public_DNS_address}
     ```

  1. Access the dashboard in your browser\. 

     ```
     http://localhost:9000
     ```

  1. Choose **Pipelines** to access the KFP UI\. 

### Create IAM Users/Roles for KFP pods and the SageMaker service<a name="create-iam-usersroles-for-kfp-pods-and-the-amazon-sagemaker-service"></a>

You now have a Kubernetes cluster with Kubeflow set up\. To run SageMaker Components for Kubeflow Pipelines, the Kubeflow Pipeline pods need access to SageMaker\. In this section, you create IAM users/roles to be used by Kubeflow Pipeline pods and SageMaker\. 

#### Create a KFP execution role<a name="create-a-kfp-execution-role"></a>

Run the following from the command line of your gateway node: 

1. Enable OIDC support on the Amazon EKS cluster with the following command\. Replace `<cluster_name>` with the name of your cluster and `<cluster_region>` with the region your cluster is in\. 

   ```
   eksctl utils associate-iam-oidc-provider --cluster <cluster-name> \
           --region <cluster-region> --approve
   ```

1. Run the following to get the [OIDC](https://openid.net/connect/) issuer URL\. This URL is in the form `https://oidc.eks.<region>.amazonaws.com/id/<OIDC_ID>`\.

   ```
   aws eks describe-cluster --region <cluster-region> --name <cluster-name> --query "cluster.identity.oidc.issuer" --output text
   ```

1. Run the following to create a file named `trust.json`\. Replace `<OIDC_URL>` with your OIDC issuer URL\. Don’t include `https://` when in your OIDC issuer URL\. Replace `<AWS_account_number>` with your AWS account number\. 

   ```
   OIDC_URL="<OIDC-URL>"
   AWS_ACC_NUM="<AWS-account-number>"
   
   # Run this to create trust.json file
   cat <<EOF > trust.json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::${AWS_ACC_NUM}:oidc-provider/${OIDC_URL}"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "${OIDC_URL}:aud": "sts.amazonaws.com",
             "${OIDC_URL}:sub": "system:serviceaccount:kubeflow:pipeline-runner"
           }
         }
       }
     ]
   }
   EOF
   ```

1. Create an IAM role named `kfp-example-pod-role` using `trust.json` using the following command\. This role is used by KFP pods to create SageMaker jobs from KFP components\. Note the ARN returned in the output\. 

   ```
   aws iam create-role --role-name kfp-example-pod-role --assume-role-policy-document file://trust.json
   aws iam attach-role-policy --role-name kfp-example-pod-role --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
   aws iam get-role --role-name kfp-example-pod-role --output text --query 'Role.Arn'
   ```

1. Edit your pipeline\-runner service account with the following command\. 

   ```
   kubectl edit -n kubeflow serviceaccount pipeline-runner
   ```

1. In the file, add the following Amazon EKS role annotation and replace `<role_arn>` with your role ARN\. 

   ```
   eks.amazonaws.com/role-arn: <role-arn>
   ```

1. Your file should look like the following when you’ve added the Amazon EKS role annotation\. Save the file\. 

   ```
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     annotations:
       eks.amazonaws.com/role-arn: <role-arn>
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"labels":{"app":"pipeline-runner","app.kubernetes.io/component":"pipelines-runner","app.kubernetes.io/instance":"pipelines-runner-0.2.0","app.kubernetes.io/managed-by":"kfctl","app.kubernetes.io/name":"pipelines-runner","app.kubernetes.io/part-of":"kubeflow","app.kubernetes.io/version":"0.2.0"},"name":"pipeline-runner","namespace":"kubeflow"}}
     creationTimestamp: "2020-04-16T05:48:06Z"
     labels:
       app: pipeline-runner
       app.kubernetes.io/component: pipelines-runner
       app.kubernetes.io/instance: pipelines-runner-0.2.0
       app.kubernetes.io/managed-by: kfctl
       app.kubernetes.io/name: pipelines-runner
       app.kubernetes.io/part-of: kubeflow
       app.kubernetes.io/version: 0.2.0
     name: pipeline-runner
     namespace: kubeflow
     resourceVersion: "11787"
     selfLink: /api/v1/namespaces/kubeflow/serviceaccounts/pipeline-runner
     uid: d86234bd-7fa5-11ea-a8f2-02934be6dc88
   secrets:
   - name: pipeline-runner-token-dkjrk
   ```

#### Create an SageMaker execution role<a name="create-an-amazonsagemaker-execution-role"></a>

The `kfp-example-sagemaker-execution-role` IAM role is used by SageMaker jobs to access AWS resources\. For more information, see the IAM Permissions section\. You provide this role as an input parameter when running the pipeline\. 

Run the following to create the role\. Note the ARN that is returned in your output\. 

```
SAGEMAKER_EXECUTION_ROLE_NAME=kfp-example-sagemaker-execution-role

TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"Service\": \"sagemaker.amazonaws.com\" }, \"Action\": \"sts:AssumeRole\" } ] }"
aws iam create-role --role-name ${SAGEMAKER_EXECUTION_ROLE_NAME} --assume-role-policy-document "$TRUST"
aws iam attach-role-policy --role-name ${SAGEMAKER_EXECUTION_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam attach-role-policy --role-name ${SAGEMAKER_EXECUTION_ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam get-role --role-name ${SAGEMAKER_EXECUTION_ROLE_NAME} --output text --query 'Role.Arn'
```

### Add access to additional IAM users or roles<a name="add-access-to-additional-iam-users-or-roles"></a>

If you use an intuitive IDE like Jupyter or want other people in your organization to use the cluster you set up, you can also give them access\. The following steps run through this workflow using SageMaker notebooks\. An SageMaker notebook instance is a fully managed Amazon EC2 compute instance that runs the Jupyter Notebook App\. You use the notebook instance to create and manage Jupyter notebooks to create ML workflows\. You can define, compile, deploy, and run your pipeline using the KFP Python SDK or CLI\. If you’re not using an SageMaker notebook to run Jupyter, you need to install the [AWS CLI ](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)and the latest version of [https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl)\. 

1. Follow the steps in [Create an SageMaker Notebook Instance](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-setup-working-env.html) to create a SageMaker notebook instance if you do not already have one\. Give the IAM role for this instance the `S3FullAccess` permission\. 

1. Amazon EKS clusters use IAM users and roles to control access to the cluster\. The rules are implemented in a config map named `aws-auth`\. Only the user/role that has access to the cluster will be able to edit this config map\. Run the following from the command line of your gateway node to get the IAM role of the notebook instance you created\. Replace `<instance-name>` with the name of your instance\. 

   ```
   aws sagemaker describe-notebook-instance --notebook-instance-name <instance-name> --region <region> --output text --query 'RoleArn'
   ```

   This command outputs the IAM role ARN in the `arn:aws:iam::<account-id>:role/<role-name>` format\. Take note of this ARN\. 

1. Run the following to attach the policies the IAM role\. Replace `<role-name>` with the `<role-name>` in your ARN\. 

   ```
   aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
   aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
   aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
   ```

1. `eksctl` provides commands to read and edit the `aws-auth` config map\. `system:masters` is one of the default user groups\. You add the user to this group\. The `system:masters` group has super user permissions to the cluster\. You can also create a group with more restrictive permissions or you can bind permissions directly to users\. Replace `<IAM-Role-arn>` with the ARN of the IAM role\. `<your_username>` can be any unique username\. 

   ```
   eksctl create iamidentitymapping \
       --cluster <cluster-name> \
       --arn <IAM-Role-arn> \
       --group system:masters \
       --username <your-username> \
       --region <region>
   ```

1. Open the Jupyter notebook on your SageMaker instance and run the following to verify that it has access to the cluster\. 

   ```
   aws eks --region <region> update-kubeconfig --name <cluster-name>
   kubectl -n kubeflow get all | grep pipeline
   ```

## Running the Kubeflow Pipeline<a name="running-the-kubeflow-pipeline"></a>

Now that setup of your gateway node and Amazon EKS cluster is complete, you can create your classification pipeline\. To create your pipeline, you need to define and compile it\. You then deploy it and use it to run workflows\. You can define your pipeline in Python and use the KFP dashboard, KFP CLI, or Python SDK to compile, deploy, and run your workflows\. The full code for the MNIST classification pipeline example is available in the [Kubeflow Github repository](https://github.com/kubeflow/pipelines/blob/master/samples/contrib/aws-samples/mnist-kmeans-sagemaker)\. To use it, clone the example Python files to your gateway node\. 

**Topics**
+ [Prepare datasets](#prepare-datasets)
+ [Create a Kubeflow Pipeline using SageMaker Components](#create-a-kubeflow-pipeline-usingamazon-sagemaker-components)
+ [Compile and deploy your pipeline](#compile-and-deploy-your-pipeline)
+ [Running predictions](#running-predictions)
+ [View results and logs](#view-results-and-logs)
+ [Cleanup](#cleanup)

### Prepare datasets<a name="prepare-datasets"></a>

To run the pipelines, you need to upload the data extraction pre\-processing script to an Amazon S3 bucket\. This bucket and all resources for this example must be located in the `us-east-1` Amazon Region\. If you don’t have a bucket, create one using the steps in [Creating a bucket](https://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html)\. 

From the `mnist-kmeans-sagemaker` folder of the Kubeflow repository you cloned on your gateway node, run the following command to upload the `kmeans_preprocessing.py` file to your Amazon S3 bucket\. Change `<bucket-name>` to the name of the Amazon S3 bucket you created\. 

```
aws s3 cp mnist-kmeans-sagemaker/kmeans_preprocessing.py s3://<bucket-name>/mnist_kmeans_example/processing_code/kmeans_preprocessing.py
```

### Create a Kubeflow Pipeline using SageMaker Components<a name="create-a-kubeflow-pipeline-usingamazon-sagemaker-components"></a>

The full code for the MNIST classification pipeline is available in the [Kubeflow Github repository](https://github.com/kubeflow/pipelines/blob/master/samples/contrib/aws-samples/mnist-kmeans-sagemaker)\. To use it, clone the example Python files to your gateway node\. 

#### Input Parameters<a name="input-parameters"></a>

The full MNIST classification pipeline has run\-specific parameters for which you must provide values when creating a run\. You must provide these parameters for each component of your pipeline\. These parameters can also be updated when using other pipelines\. We have provided default values for all parameters in the sample classification pipeline file\. 

The following are the only parameters you need to pass to run the sample pipelines\. To pass these parameters, update their entries when creating a new run\. 
+ **Role\-ARN:** This must be the ARN of an IAM role that has full SageMaker access in your AWS account\. Use the ARN of  `kfp-example-pod-role`\. 
+ **Bucket**: This is the name of the Amazon S3 bucket that you uploaded the `kmeans_preprocessing.py` file to\. 

You can adjust any of the input parameters using the KFP UI and trigger your run again\. 

### Compile and deploy your pipeline<a name="compile-and-deploy-your-pipeline"></a>

After defining the pipeline in Python, you must compile the pipeline to an intermediate representation before you can submit it to the Kubeflow Pipelines service\. The intermediate representation is a workflow specification in the form of a YAML file compressed into a tar\.gz file\. You need the KFP SDK to compile your pipeline\. 

#### Install KFP SDK<a name="install-kfp-sdk"></a>

Run the following from the command line of your gateway node: 

1. Install the KFP SDK following the instructions in the [Kubeflow pipelines documentation](https://www.kubeflow.org/docs/pipelines/sdk/install-sdk/)\. 

1. Verify that the KFP SDK is installed with the following command: 

   ```
   pip show kfp
   ```

1. Verify that `dsl-compile` has been installed correctly as follows: 

   ```
   which dsl-compile
   ```

#### Compile your pipeline<a name="compile-your-pipeline"></a>

You have three options to interact with Kubeflow Pipelines: KFP UI, KFP CLI, or the KFP SDK\. The following sections illustrate the workflow using the KFP UI and CLI\. 

Complete the following from your gateway node to compile your pipeline\. 

1. Modify your Python file with your Amazon S3 bucket name and IAM role ARN\. 

1. Use the `dsl-compile` command from the command line to compile your pipeline as follows\. Replace `<path-to-python-file>` with the path to your pipeline and `<path-to-output>` with the location where you want your tar\.gz file to be\. 

   ```
   dsl-compile --py <path-to-python-file> --output <path-to-output>
   ```

#### Upload and run the pipeline using the KFP CLI<a name="upload-and-run-the-pipeline-using-the-kfp-cli"></a>

Complete the following steps from the command line of your gateway node\. KFP organizes runs of your pipeline as experiments\. You have the option to specify an experiment name\. If you do not specify one, the run will be listed under **Default** experiment\. 

1. Upload your pipeline as follows: 

   ```
   kfp pipeline upload --pipeline-name <pipeline-name> <path-to-output-tar.gz>
   ```

   Your output should look like the following\. Take note of the `ID`\. 

   ```
   Pipeline 29c3ff21-49f5-4dfe-94f6-618c0e2420fe has been submitted
   
   Pipeline Details
   ------------------
   ID           29c3ff21-49f5-4dfe-94f6-618c0e2420fe
   Name         sm-pipeline
   Description
   Uploaded at  2020-04-30T20:22:39+00:00
   ...
   ...
   ```

1. Create a run using the following command\. The KFP CLI run command currently does not support specifying input parameters while creating the run\. You need to update your parameters in the Python pipeline file before compiling\. Replace `<experiment-name>` and `<job-name>` with any names\. Replace `<pipeline-id>` with the ID of your submitted pipeline\. Replace `<your-role-arn>` with the ARN of `kfp-example-pod-role`\. Replace `<your-bucket-name>` with the name of the Amazon S3 bucket you created\. 

   ```
   kfp run submit --experiment-name <experiment-name> --run-name <job-name> --pipeline-id <pipeline-id> role_arn="<your-role-arn>" bucket_name="<your-bucket-name>"
   ```

   You can also directly submit a run using the compiled pipeline package created as the output of the `dsl-compile` command\. 

   ```
   kfp run submit --experiment-name <experiment-name> --run-name <job-name> --package-file <path-to-output> role_arn="<your-role-arn>" bucket_name="<your-bucket-name>"
   ```

   Your output should look like the following: 

   ```
   Creating experiment aws.
   Run 95084a2c-f18d-4b77-a9da-eba00bf01e63 is submitted
   +--------------------------------------+--------+----------+---------------------------+
   | run id                               | name   | status   | created at                |
   +======================================+========+==========+===========================+
   | 95084a2c-f18d-4b77-a9da-eba00bf01e63 | sm-job |          | 2020-04-30T20:36:41+00:00 |
   +--------------------------------------+--------+----------+---------------------------+
   ```

1. Navigate to the UI to check the progress of the job\. 

#### Upload and run the pipeline using the KFP UI<a name="upload-and-run-the-pipeline-using-the-kfp-ui"></a>

1. On the left panel, choose the **Pipelines** tab\. 

1. In the upper\-right corner, choose **\+UploadPipeline**\. 

1. Enter the pipeline name and description\. 

1. Choose **Upload a file** and enter the path to the tar\.gz file you created using the CLI or with the Python SDK\. 

1. On the left panel, choose the **Pipelines** tab\. 

1. Find the pipeline you created\. 

1. Choose **\+CreateRun**\. 

1. Enter your input parameters\. 

1. Choose **Run**\. 

### Running predictions<a name="running-predictions"></a>

Once your classification pipeline is deployed, you can run classification predictions against the endpoint that was created by the Deploy component\. Use the KFP UI to check the output artifacts for `sagemaker-deploy-model-endpoint_name`\. Download the \.tgz file to extract the endpoint name or check the SageMaker console in the region you used\. 

#### Configure permissions to run predictions<a name="configure-permissions-to-run-predictions"></a>

If you want to run predictions from your gateway node, skip this section\. 

1. To use any other machine to run predictions, assign the `sagemaker:InvokeEndpoint` permission to the IAM role or IAM user used by the client machine\. This permission is used to run predictions\. 

1. On your gateway node, run the following to create a policy file: 

   ```
   cat <<EoF > ./sagemaker-invoke.json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "sagemaker:InvokeEndpoint"
               ],
               "Resource": "*"
           }
       ]
   }
   EoF
   ```

1. Attach the policy to the client node’s IAM role or IAM user\. 

1. If your client machine has an IAM role attached, run the following\. Replace `<your-instance-IAM-role>` with the name of the client node’s IAM role\. Replace `<path-to-sagemaker-invoke-json>` with the path to the policy file you created\. 

   ```
   aws iam put-role-policy --role-name <your-instance-IAM-role> --policy-name sagemaker-invoke-for-worker --policy-document file://<path-to-sagemaker-invoke-json>
   ```

1. If your client machine has IAM user credentials configured, run the following\. Replace `<your_IAM_user_name>` with the name of the client node’s IAM user\. Replace `<path-to-sagemaker-invoke-json>` with the path to the policy file you created\. 

   ```
   aws iam put-user-policy --user-name <your-IAM-user-name> --policy-name sagemaker-invoke-for-worker --policy-document file://<path-to-sagemaker-invoke-json>
   ```

#### Run predictions<a name="run-predictions"></a>

1. Create a Python file from your client machine named `mnist-predictions.py` with the following content\. Replace the `ENDPOINT_NAME` variable\. This script loads the MNIST dataset, then creates a CSV from those digits and sends it to the endpoint for prediction\. It then outputs the results\.

   ```
   import boto3
   import gzip
   import io
   import json
   import numpy
   import pickle
   
   ENDPOINT_NAME='<endpoint-name>'
   region = boto3.Session().region_name
   
   # S3 bucket where the original mnist data is downloaded and stored
   downloaded_data_bucket = f"jumpstart-cache-prod-{region}"
   downloaded_data_prefix = "1p-notebooks-datasets/mnist"
   
   # Download the dataset
   s3 = boto3.client("s3")
   s3.download_file(downloaded_data_bucket, f"{downloaded_data_prefix}/mnist.pkl.gz", "mnist.pkl.gz")
   
   # Load the dataset
   with gzip.open('mnist.pkl.gz', 'rb') as f:
       train_set, valid_set, test_set = pickle.load(f, encoding='latin1')
   
   # Simple function to create a csv from our numpy array
   def np2csv(arr):
       csv = io.BytesIO()
       numpy.savetxt(csv, arr, delimiter=',', fmt='%g')
       return csv.getvalue().decode().rstrip()
   
   runtime = boto3.Session(region).client('sagemaker-runtime')
   
   payload = np2csv(train_set[0][30:31])
   
   response = runtime.invoke_endpoint(EndpointName=ENDPOINT_NAME,
                                      ContentType='text/csv',
                                      Body=payload)
   result = json.loads(response['Body'].read().decode())
   print(result)
   ```

1. Run the Python file as follows: 

   ```
   python mnist-predictions.py
   ```

### View results and logs<a name="view-results-and-logs"></a>

When the pipeline is running, you can choose any component to check execution details, such as inputs and outputs\. This lists the names of created resources\. 

If the KFP request is successfully processed and an SageMaker job is created, the component logs in the KFP UI provide a link to the job created in SageMaker\. The CloudWatch logs are also provided if the job is successfully created\. 

If you run too many pipeline jobs on the same cluster, you may see an error message that indicates you do not have enough pods available\. To fix this, log in to your gateway node and delete the pods created by the pipelines you are not using as follows: 

```
kubectl get pods -n kubeflow
kubectl delete pods -n kubeflow <name-of-pipeline-pod>
```

### Cleanup<a name="cleanup"></a>

When you’re finished with your pipeline, you need to clean up your resources\. 

1. From the KFP dashboard, terminate your pipeline runs if they do not exit properly by choosing **Terminate**\. 

1. If the **Terminate** option doesn’t work, log in to your gateway node and manually terminate all the pods created by your pipeline run as follows: 

   ```
   kubectl get pods -n kubeflow
   kubectl delete pods -n kubeflow <name-of-pipeline-pod>
   ```

1. Using your AWS account, log in to the SageMaker service\. Manually stop all training, batch transform, and HPO jobs\. Delete models, data buckets, and endpoints to avoid incurring any additional costs\. Terminating the pipeline runs does not stop the jobs in SageMaker\. 