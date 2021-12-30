# Setting Up Neptune SageMaker Notebook

### Prerequisites
---
>- You will need to already have a Neptune cluster set up
>- For our Neptune cluster, we have IAM db authentication disabled, encyrption enabled, engine version 1.0.4.1

### Enabling Neptune ML via CloudFormation
---
1. Launch the `NeptuneSagemakerWithTOC.yaml` file found in this directory to create most of the necessary resources for SageMaker and NeptuneML enablement. These resources also include enabling Table of Contents and a Git Codecommit Repository for the Sagemaker notebook. (The `NeptuneSagemaker.yaml` file is stored here for those that do not wish to enable Table of Contents or the Git Repository.) This is what the parameters should look like after successfully launching the stack.

    Many of these parameters were created in the Neptune Cluster set up. You can look at the cluster configuration/cloudformation outputs to find them. Also, make sure to read the descriptions of each parameter as they will give you detailed information.

    **Important to note that the codecommit zip file should contain any code you would like to access from your Sagemaker notebook. You should upload this file to the S3 Bucket you reference as the 'InitialCommitBucket'.


    Now, you will need to do a few things manually in the console. 

2. First, you'll need to go to the VPC console and click on Security Groups. Check that the 3 security groups created by the CloudFormation stack exist, and ensure that they all have these 2 inbound rules 

    If any of those security groups do not have those inbound rules, add them now. 

3. Then, go to the Neptune console and click on your cluster name. Then click on the parameter group associated with the cluster. 


    Ensure that the parameter group has the correct `neptune_lab_mode`, `neptune_ml_endpoint`, `neptune_ml_iam_role`, and `neptune_query_timeout` values (you can check if these are correct by referencing your CloudFormation stack). 

    **Note**: Ensure that you provide the Arn for the IAM role.

    We plan to check to see if the `neptune_ml_endpoint` is necessary, but for now use the EndpointID of the 'sagemaker.runtime' endpoint.

4. If they are correct, then go back to your Neptune cluster, and click `Actions` in the top right. Then click `Manage IAM Roles`. Make sure the correct Neptune ML IAM Role is in there. 


    If it isn't, add the correct one now. You can find the correct IAM role by referencing your CloudFormation stack.

5. You will now need to reboot the Neptune cluster to ensure that the updated parameter group has been correctly associated to the cluster. You can do this by `stopping` and `starting` your cluster. This will take approximately 20-30 minutes. When your cluster has rebooted, it will be Neptune Enabled.

6. Now you are able to go into the SageMaker console and open your notebook instance.  You can now use any of the prepopulated notebooks for machine learning or create your own. Happy coding!

    - If you chose the file that enables a Table of Contents within your Sagemaker notebook do the following steps upon opening your notebook:

        - Wait for the tab in the top of the screen for `nbextensions` to pop up. Once that has loaded, click on it, then enable the TOC by selecting `Table of Contents` and saving. 

### Create An Extra Sagemaker NeptuneML Notebook
---

If you have already enabled NeptuneML, but would like a quick and easy way to launch another enabled Sagemaker notebook, you can use the Cloudformation template titled `NotebookCreationOnlywGithub.yaml`. You can fill out the parameters from the parameters and outputs of the nested NeptuneML Enablement cloudformation.

### SageMaker NeptuneML Notebook Code
---

See the .ipynb files in /crg-ml-models for our most up-to-date SageMaker ML work. We have the iteration 1,2,&3 of the first ML Insights Ec2 use case. We use relational dataframe ML algorithms to predict which Ec2's have the riskiest workload. You can also take a look at this SageMaker notebook by going to the AWS SageMaker console, clicking on 'Notebook Instances', then open one of the CRG ML sagemaker juptyer notebooks. Once you are in the notebook, you can find these files within the designated folder which is also associated to a codecommit git repository.
