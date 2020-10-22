# Automating QuickSight Enablement

In this section, we will explain how to use CloudFormation and CodeBuild to automate the process of creating and loading data into to a database, and connecting that database to QuickSight to use as a data source. 

---
## Prerequisites

Before we can create and load the database into QuickSight, we must have a few resources set up. We assume that you already have a VPC and VPC Subnet groups created. If you don't, please see these resources to set up a [VPC](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/gsg_create_vpc.html) and a [Subnet Group](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.create-cluster.console.create-subnet-group.html)
1. *S3 Bucket*: In this section we will use Boto3 to interact with the Amazon S3 API and create a S3 bucket. We will need a specific S3 bucket so that we can load our template into it for retrieval via CloudFormation. Within your Cloud9 virtual environment, be sure to install `awscli` and `boto3`.

    ```bash
    pip3 install awscli
    pip3 install boto3
    ```
    Create a python file to load the following script. This script will create the bucket and encrypt the bucket.

    ```py
    import boto3

    s3 = boto3.client('s3')
    sts = boto3.client('sts')

    awsAccountId = sts.get_caller_identity()['Account']
    bucketName = 'quicksight-enablement' + awsAccountId

    response = s3.create_bucket(
        ACL='private',
        Bucket = bucketName
        )
    print(response)

    response = s3.list_buckets()
    try:
        for bucket in response['Buckets']:
            bucketName = str(bucket['Name'])
            response = s3.get_bucket_encryption(Bucket=bucketName)
    except Exception as e:
        if str(e) == 'An error occurred (ServerSideEncryptionConfigurationNotFoundError) when calling the GetBucketEncryption operation: The server side encryption configuration was not found':
            response = s3.put_bucket_encryption(
                Bucket=bucketName,
                ServerSideEncryptionConfiguration={
                    'Rules': [
                        {
                            'ApplyServerSideEncryptionByDefault': {
                                'SSEAlgorithm': 'AES256'
                            }
                        }
                    ]
                }
            )
            print(response)
            print('Encryption applied to ' + bucketName)
        else:
            print(e)
            raise
    ```

    The `bucketName` must be a unique value, hence why `awsAccountId` variable is introduced. You can replace 'quicksight-enablement' with your specific bucket name. After executing the file, you can use the below command to retrieve the full version of the generated `bucketName` once its created. This command returns a list of all S3 buckets, so you can pinpoint your S3 bucket's unique name from this list.

    ```bash
    aws s3api list-buckets --query "Buckets[].Name"
    ```
2. *JSON Data to S3 Bucket*: Now that we have the S3 bucket available, we can load the data that we would like loaded into QuickSight. 

    Execute these lines on your command line in Cloud9 environment. This will copy the JSON file (we use guardduty-port-findings.json) from Azure DevOps to your s3 bucket created for this project. Before running these lines, ensure that you have inserted the correct directory, file name, and bucket name. 

    ```bash
    git clone https://github.com/osamples/CloudSecurityEngineering.git/Amazon-QuickSight-PostgreSQL-Automation
    cd Amazon-QuickSight-PostgreSQL-Automation/Sample-Datasets
    aws s3 cp guardduty-port-findings.json s3://YOUR_BUCKET_NAME
    ```


3. *Security Groups*: Before we can make the connection to QuickSight, we must [authorize it](https://docs.aws.amazon.com/quicksight/latest/user/enabling-access-rds.html).

    In other words, we are creating a new Security Group.

    Within the AWS VPC console under `Network & Security`, select `Security Groups` and then `Create security group`.

    ![CreateSecurityGroup](./screenshots/CreateSecurityGroup.png)

    Name the security group something like `'QuickSight-Access'`, provide a description, and choose the same VPC that your database uses.

    Add an inbound rule, choose `'PostgreSQL'` for `'Type'`, any description, and keep `'Source'` as `'Custom'`. The value for the custom source should associate to the region in which your database is in. You can find the correct CIDR [here](https://docs.aws.amazon.com/quicksight/latest/user/regions.html).

    ![CreateSecurityGroup2](./screenshots/CreateSecurityGroup2.png)

    Create the Security Group.

    Next, we will create a security group to access the database (for us, this was `DevSecOps-QuickSightEnablement`). We will have to add inbound rules in this security group as well.

    First, select inbound rules. Add two rules with `Port range` = `0-65535`, and the sources as the security group you created for QuickSight access at the beginning of this section (Amazon-QuickSight-access) along with the one we are creating now ('DevSecOps-QuickSightEnablement').
    Then add Cloud9 Security Group sources for those who would like to access the database (for us, Jonathan, Olivia, and Katie had access). Be sure to specify 'PostgreSQL' as the type. 


    Once you are done with this, your configuration should look like this. 

    ![SecurityGroup](./screenshots/databasesecuritygroup.png)

    Now, go back to the 'QuickSight Access' Security group and add another inbound rule for the DevSecOps-QuickSight Enablement group we just created. When you're done updating the Security Group, it should look like this.

    ![QSInbound](./screenshots/QSInbound.png)

4. *QuickSight VPC Connection*: Now go to the AWS QuickSight console. Click on the user at the top right corner and select `'Manage QuickSight'`. On the left-hand side of the page, select `'Manage VPC connections'`. We will need to add a VPC connection specific to the Security Group we just created for `'QuickSight Access'`.

    ![ManageQuickSight](./screenshots/ManageQuickSight.png)

    Select `'Add VPC Connection'`, create a name for it, choose the same VPC (e.g. DevSecOps) that your database uses, choose the same Subnet ID that was used when filling out the parameters for your database (e.g. us-east-1d), and paste the Security group ID from the security group created for QuickSight access (for us this is `sg-084ad5f1763e1c260`). Click `'Create'`.

    ![AddVPCConnection](./screenshots/AddVPCConnection.png)
    ![VPCConfiguration](./screenshots/VPCConfiguration.png)

## Steps for CloudFormation and Codebuild 
This section describes how to deploy the automated process. You can skip to Step 5 if you are only changing parameters, such as the JSON file.

1.  Create a file named `QuickSight_PostgreSQL_Dataset_CodeBuild_CloudFormation.yaml`. This file will be reponsible for:
    - Creating a Secret
    - Creating a DB Cluster
    - Creating a CodeCommit repo 
    - Creating IAM roles for the DB and the CodeBuild Event
    - Creating a CodeBuild project 
    - Creating CodeBuild Event rules
         

    Paste the following contents into the yaml file you just created:

    ```yml
    AWSTemplateFormatVersion: 2010-09-09
    Description: Deploys resources needed to create Aurora PostgreSQL Serverless DB, connect to and load files into the database from S3, and then create a QuickSight Data Source from the raw information. THIS IS A PROOF OF CONCEPT, replace your raw data as needed.
    Parameters:
        DatabaseName:    
            Type: String    
            Description: Unique name to call Database Instance  
        SecretName:
            Type: String
            Description: Unique name to call Secret for SecretsManager
        EngineVersion:    
            Type: String    
            Description: Database engine version. Version 10.7 is compatible with PostgreSQL Serverless.    
            Default: '10.7' 
        AvailabilityZone:    
            Type: String    
            Description: Please ensure that this zone matches one of the subnet regions chosen.     
            Default: us-east-1d  
        VpcId:    
            Type: AWS::EC2::VPC::Id    
            Description: Choose your specific VPC.  
        VpcSubnetId:    
            Type: AWS::EC2::Subnet::Id    
            Description: Ensure the subnet region correlates to the Availability Zone specified and is Private.
        VpcSubnetId2:    
            Type: AWS::EC2::Subnet::Id    
            Description: Ensure the subnet region is Private.
        VpcSecurityGroupId:    
            Type: AWS::EC2::SecurityGroup::Id    
            Description: Choose the Security Group created specifically for the database instance.  
        DBSubnetGroupName:    
            Type: String    
            Description: This is the unique VPC ID.
        BackupRetentionPeriod:    
            Type: Number    
            Description: The number of days for which automated backups are retained.    
            Default: 1  
        DeletionProtection:    
            Type: String    
            Default: True    
            AllowedValues:       
                - True      
                - False    
            Description: A value that indicates whether the DB cluster has deletion protection enabled. The database can't be deleted when deletion protection is enabled. By default, the deletion protection is enabled.
        FileName:
            Type: String
            Description: Name of JSON file that you would like to load into DB and QuickSight
            Default: guardduty-port-findings.json
        CodeBuildProjName:
            Type: String
            Description: Name for your CodeBuild Project
            Default: QuickSight-Aurora-PostgreSQL-DataSource-Automation
        QuickSightDataSource:
            Type: String
            Description: The name for the QuickSight Data Source you will create
            Default: aurora-postgresql-json-datasource
        QuickSightGroupName:
            Type: String
            Description: The name for the QuickSight Group you will create
            Default: InfosecAdmins
        VPCConnectionArn:
            Type: String
            Description: The ARN for your specific VPC Connection
        InitialCommitBucket:
            Type: String
            Description: The name of the S3 bucket you uploaded the JSON file and ZIP archive of the code artifacts to 
        InitialCommitKey:
            Type: String
            Description: Name of the package for the initial commit for the DevSecOps pipeline DO NOT include .zip
            Default: codecommit-archive
    Resources:
        MySecret:
            Type: 'AWS::SecretsManager::Secret'
            Properties:
            Name: !Ref SecretName
            Description: "This secret has a dynamically generated secret password."
            GenerateSecretString:
                SecretStringTemplate: '{"username": "postgres"}'
                GenerateStringKey: "password"
                PasswordLength: 30
                ExcludeCharacters: '"@/\'
            Tags:
                -
                Key: 
                Value: 
        Cluster:
            Type: AWS::RDS::DBCluster
            Properties:
            AvailabilityZones: 
                - !Ref AvailabilityZone
            Engine: aurora-postgresql
            EngineMode: serverless
            Port: 5432
            DBClusterParameterGroupName: default.aurora-postgresql10
            SourceRegion: !Ref AvailabilityZone
            EngineVersion: !Ref EngineVersion
            DatabaseName: !Ref DatabaseName
            MasterUsername: !Sub '{{resolve:secretsmanager:${MySecret}::username}}'
            MasterUserPassword: !Sub '{{resolve:secretsmanager:${MySecret}::password}}'
            DBClusterIdentifier: !Ref AWS::StackName
            BackupRetentionPeriod: !Ref BackupRetentionPeriod
            DeletionProtection: !Ref DeletionProtection
            EnableHttpEndpoint: True
            VpcSecurityGroupIds:
                - !Ref VpcSecurityGroupId
            DBSubnetGroupName: !Ref DBSubnetGroupName
            Tags:
                -
                Key: 
                Value: 
        QuickSightDataSetCommit:
            Type: AWS::CodeCommit::Repository
            Properties:
            RepositoryDescription: Contains the code artifacts for CodeBuild to create QuickSight Datasets - Managed by CloudFormation
            RepositoryName: quicksight-aurora-postgresql-repo
            Code:
                S3:
                Bucket: !Ref InitialCommitBucket
                Key: !Sub '${InitialCommitKey}.zip'
            Tags:
                -
                Key:
                Value:
        CodeBuildServiceRole:
            Type: AWS::IAM::Role
            Properties:
            ManagedPolicyArns:
                - arn:aws:iam::aws:policy/AmazonEC2FullAccess
            Policies:
            - PolicyName: CodeBuildServiceRolePolicy
                PolicyDocument:
                Version: 2012-10-17
                Statement:
                - Effect: Allow
                    Action:
                    - codecommit:GitPull
                    Resource: !GetAtt QuickSightDataSetCommit.Arn
                - Effect: Allow
                    Action:
                    - s3:GetObject
                    - s3:GetObjectVersion
                    - s3:PutObject
                    - s3:GetBucketAcl
                    - s3:GetBucketLocation
                    Resource:
                    - !Sub 'arn:aws:s3:::${InitialCommitBucket}'
                    - !Sub 'arn:aws:s3:::${InitialCommitBucket}/*'          
                - Effect: Allow
                    Action:
                    - kms:Decrypt
                    - logs:CreateLogGroup
                    - logs:CreateLogStream
                    - logs:PutLogEvents
                    - quicksight:*
                    - iam:PassRole
                    Resource: '*'
                - Effect: Allow
                    Action:
                    - secretsmanager:GetResourcePolicy
                    - secretsmanager:GetSecretValue
                    - secretsmanager:DescribeSecret
                    - secretsmanager:ListSecretVersionIds
                    Resource:
                    - !Ref MySecret
                - Effect: Allow
                    Action:
                    - secretsmanager:GetRandomPassword
                    - secretsmanager:ListSecrets
                    Resource: '*'
            AssumeRolePolicyDocument:
                Version: 2012-10-17
                Statement:
                - Effect: Allow
                Principal: { Service: codebuild.amazonaws.com }
                Action:
                - sts:AssumeRole
            Tags:
                -
                Key:
                Value:
        QuickSightDataSetCodeBuild:
            Type: AWS::CodeBuild::Project
            Properties:
            VpcConfig:
                SecurityGroupIds: 
                - !Ref VpcSecurityGroupId
                Subnets: 
                - !Ref VpcSubnetId
                - !Ref VpcSubnetId2
                VpcId: !Ref VpcId
            Name: !Ref CodeBuildProjName
            Description: Connects to Aurora PostgreSQL Serverless database, loads JSON data into database, and creates a QuickSight datasource.  - Managed by CloudFormation
            Environment:
                ComputeType: BUILD_GENERAL1_MEDIUM
                Image: aws/codebuild/standard:4.0
                PrivilegedMode: True
                Type: LINUX_CONTAINER
                EnvironmentVariables:
                - Name: MY_SECRET
                Type: PLAINTEXT
                Value: !Ref SecretName
                - Name: DB_ENDPOINT
                Type: PLAINTEXT
                Value: !GetAtt Cluster.Endpoint.Address
                - Name: DB_NAME
                Type: PLAINTEXT
                Value: !Ref DatabaseName
                - Name: S3_FILENAME
                Type: PLAINTEXT
                Value: !Ref FileName
                - Name: QUICKSIGHT_GROUP_NAME
                Type: PLAINTEXT
                Value: !Ref QuickSightGroupName
                - Name: QUICKSIGHT_DATASOURCE_BUCKET
                Type: PLAINTEXT
                Value: !Ref InitialCommitBucket
                - Name: QUICKSIGHT_DATASOURCE_NAME
                Type: PLAINTEXT
                Value: !Ref QuickSightDataSource
                - Name: VPC_CONNECTION_ARN
                Type: PLAINTEXT
                Value: !Ref VPCConnectionArn
            LogsConfig:
                CloudWatchLogs:
                Status: ENABLED
            ServiceRole: !GetAtt CodeBuildServiceRole.Arn
            Artifacts:
                Type: NO_ARTIFACTS
            Source:
                BuildSpec: buildspec.yml
                Type: CODECOMMIT
                Location: !GetAtt QuickSightDataSetCommit.CloneUrlHttp
            Tags:
                -
                Key:
                Value:
        CodeBuildEventRole:
            Type: AWS::IAM::Role
            Properties:
            RoleName: QuickSight-Aurora-PostgreSQL-DataSet-Creator-Sched-Role
            Policies:
            - PolicyName: QuickSight-Aurora-PostgreSQL-DataSet-Creator-CodeBuildServiceRolePolicy
                PolicyDocument:
                Version: 2012-10-17
                Statement:
                - Effect: Allow
                    Action:
                    - codebuild:StartBuild
                    Resource: '*'
            AssumeRolePolicyDocument:
                Version: 2012-10-17
                Statement:
                - Effect: Allow
                Principal: { Service: events.amazonaws.com }
                Action:
                    - sts:AssumeRole
            Tags:
                -
                Key:
                Value:
        QuickSightDataSetCodeBuildEvent: 
            Type: AWS::Events::Rule
            Properties:
            Name: QuickSight-Aurora-PostgreSQL-DataSet-Creator-Rule
            Description: Runs QuickSight-Aurora-PostgreSQL-DataSet-Creator every 24 hours - Managed by CloudFormation
            ScheduleExpression: rate(24 hours)
            State: ENABLED
            Targets: 
                - 
                Arn: !GetAtt QuickSightDataSetCodeBuild.Arn
                Id: QSCBAutomationTrigger
                RoleArn: !GetAtt CodeBuildEventRole.Arn
    ```
2.  Create a file named `dbconnect.py` with the following contents. This file is responsible for using psycopg2 to connect to the database created in the CloudFormation template, loading data into the database, creating the data source in QuickSight as well as the data group in QuickSight.

    ```py
    import psycopg2
    import os
    import boto3
    import json
    import base64
    import botocore
    from botocore.exceptions import ClientError

    # Import Boto3 Clients
    sts = boto3.client('sts')
    s3 = boto3.client('s3')
    quicksight = boto3.client('quicksight')
    secretsmanager = boto3.client('secretsmanager')

    # Set Global Variables and Lists
    awsRegion = os.environ['AWS_REGION']
    awsAccountId = sts.get_caller_identity()['Account']
    secretValue = os.environ['MY_SECRET']
    hostName = os.environ['DB_ENDPOINT']
    portNumber = 5432
    dbName = os.environ['DB_NAME']
    fileName = os.environ['S3_FILENAME']
    groupName = os.environ['QUICKSIGHT_GROUP_NAME']
    reportBucket = os.environ['QUICKSIGHT_DATASOURCE_BUCKET']
    dataSourceName = os.environ['QUICKSIGHT_DATASOURCE_NAME']
    vpcConnectionArn = os.environ['VPC_CONNECTION_ARN']

    #Get User and Password
    response = secretsmanager.get_secret_value(SecretId=secretValue)
    dbSecret = json.loads(response['SecretString'])

    dbUser = str(dbSecret['username'])
    dbPw = str(dbSecret['password'])

    #Load in file from S3 Bucket as 'file.json'
    try:
        save_as = './file.json'
        s3.download_file(reportBucket, fileName, save_as)
        
    except Exception as e:
        print(e)
        raise

    #Connect to and load data into DB
    def connect_load_data():
        try:
            connection = psycopg2.connect(user=dbUser,password=dbPw,host=hostName,port=portNumber,database=dbName)
        
            cursor = connection.cursor()
        # Print PostgreSQL Connection properties
            print ( connection.get_dsn_parameters(),"\n")
        
        # Print PostgreSQL version
            cursor.execute("SELECT version();")
            record = cursor.fetchone()
            print("You are connected to - ", record,"\n")
            
        ########### https://kb.objectrocket.com/postgresql/insert-json-data-into-postgresql-using-python-part-2-1248
            import json,sys
            from psycopg2 import connect, Error
            with open(save_as) as json_data:
                record_list = json.load(json_data)
                
            if type(record_list) == list:
                columns = [list(x.keys()) for x in record_list][0]
                print('\ncolumn names:', columns)
                table_name = "json_data"
                sql_string = 'INSERT INTO {} '.format( table_name )
                sql_string += "(" + ', '.join(columns) + ")\nVALUES "
                
                #This creates a table with all VARCHAR datatypes.
                create_table = 'CREATE TABLE {} '.format( table_name )
                create_table += "(" + ' VARCHAR, '.join(columns) + " VARCHAR );"
                
                #Use the below line to create a table with specific datatypes for your respective columns.
                #create_table = 'CREATE TABLE {} '.format( table_name) + '(AwsAccountId VARCHAR, AwsRegion VARCHAR, InstanceId VARCHAR, InstanceArn VARCHAR, FindingType VARCHAR, FindingTimestamp TIMESTAMP, RemoteIpv4Address VARCHAR, RemotePortNumber Integer, RemotePortName VARCHAR, LocalPortNumber Integer, LocalPortName VARCHAR, IpProtocol VARCHAR, Longitude NUMERIC, Latitude NUMERIC);'
            
        # enumerate over the record
            for i, record_dict in enumerate(record_list):
            
                # iterate over the values of each record dict object
                values = []
                for col_names, val in record_dict.items():
            
                    # Postgres strings must be enclosed with single quotes
                    if type(val) == str:
                        # escape apostrophies with two single quotations
                        val = val.replace("'", "''")
                        val = "'" + val + "'"
            
                    values += [ str(val) ]
                    
                        # join the list of values and enclose record in parenthesis
                sql_string += "(" + ', '.join(values) + "),\n"
            
            # remove the last comma and end statement with a semicolon
            sql_string = sql_string[:-2] + ";"
        
            # only attempt to execute SQL if cursor is valid
            if cursor != None:
            
                try:
                    cursor.execute(create_table)
                    cursor.execute( sql_string )
                    connection.commit()
            
                    print ('\nfinished INSERT INTO execution')
            
                except (Exception, Error) as error:
                    print("\nexecute_sql() error:", error)
                    connection.rollback()
            
                # close the cursor and connection
                cursor.close()
                connection.close()
                print("PostgreSQL connection is closed")
        ##################
        
        except (Exception, psycopg2.Error) as error :
            print ("Error while connecting to PostgreSQL", error)
            raise error

    def create_quicksight_group():
        try:
            response = quicksight.create_group(
                GroupName=groupName,
                Description='Dynamically created Group that contains all QuickSight Users currently onboarded',
                AwsAccountId=awsAccountId,
                Namespace='default' # this MUST be 'default'
            )
            groupPrincipalArn = str(response['Group']['Arn'])
            print(groupName + ' was created succesfully')
            print(groupName + ' ARN is ' + groupPrincipalArn)
        except botocore.exceptions.ClientError as error:
            # If the Group exists already, handle the error gracefull
            if error.response['Error']['Code'] == 'ResourceExistsException':
                response = quicksight.describe_group(
                    GroupName=groupName,
                    AwsAccountId=awsAccountId,
                    Namespace='default' # this MUST be 'default'
                )
                groupArn = str(response['Group']['Arn'])
                print('A Group with the name ' + groupName + ' already exists! Attempting to add Users into it')
                print('As a reminder the ARN for ' + groupName + ' is: ' + groupArn)
            else:
                raise error
        
        try:
            response = quicksight.list_users(
                AwsAccountId=awsAccountId,
                MaxResults=100,
                Namespace='default' # this MUST be 'default'
            )
            for u in response['UserList']:
                userName = str(u['UserName'])
                roleLevel = str(u['Role'])
                if roleLevel == 'ADMIN' or 'AUTHOR':
                    quicksight.create_group_membership(
                        MemberName=userName,
                        GroupName=groupName,
                        AwsAccountId=awsAccountId,
                        Namespace='default' # this MUST be 'default'
                    )
                    print('User ' + userName + ' added to Group ' + groupName)
                else:
                    pass
        except Exception as e:
            print(e)

    def create_quicksight_datasource():
        try:
            quicksight.create_data_source(
                AwsAccountId=awsAccountId,
                DataSourceId=dataSourceName,
                Name=dataSourceName,
                Type='AURORA_POSTGRESQL',
                VpcConnectionProperties={
                    'VpcConnectionArn': vpcConnectionArn
                },
                Permissions=[
                    {
                        'Principal': 'arn:aws:quicksight:' + awsRegion + ':' + awsAccountId + ':group/default/' + groupName,
                        'Actions': [
                            'quicksight:DescribeDataSource',
                            'quicksight:DescribeDataSourcePermissions',
                            'quicksight:PassDataSource',
                            'quicksight:UpdateDataSource',
                            'quicksight:DeleteDataSource',
                            'quicksight:UpdateDataSourcePermissions'
                        ]
                    }
                ],
                DataSourceParameters={
                    'AuroraPostgreSqlParameters': {
                        'Host': hostName,
                        'Port': portNumber,
                        'Database': dbName
                    } 
                },
                Credentials={
                    'CredentialPair': {
                        'Username': dbUser,
                        'Password': dbPw,
                    }
                }
            )
            print('Data Source ' + dataSourceName + ' was created')
        except botocore.exceptions.ClientError as error:
            # If the Group exists already, handle the error gracefull
            if error.response['Error']['Code'] == 'ResourceExistsException':
                print('The Data Source ' + dataSourceName + ' already exists, attempting to update it')
                quicksight.update_data_source(
                    AwsAccountId=awsAccountId,
                    DataSourceId=dataSourceName,
                    Name=dataSourceName,
                    VpcConnectionProperties={
                        'VpcConnectionArn': vpcConnectionArn
                    },
                    DataSourceParameters={
                        'AuroraPostgreSqlParameters': {
                            'Host': hostName,
                            'Port': portNumber,
                            'Database': dbName
                        } 
                    }
                )
                print('Data Source ' + dataSourceName + ' was updated')
            else:
                raise error
                
    def main():
        connect_load_data()
        create_quicksight_group()
        create_quicksight_datasource()
        
    main()
    ```

3.  Create a file named `buildspec.yml` with the following contents. This file will be responsible for telling CodeBuild what sequence of events to follow (installing the correct packages, and executing the `dbconnect.py` file, in this case).

    ```yml
    version: 0.2

    phases:
    install:
        commands:
        - apt update
        - pip3 install --upgrade pip
        - pip3 install boto3
        - pip3 install psycopg2
    build:
        commands:
        - python3 dbconnect.py
        - echo GuardDuty JSON file loaded into Aurora PostgreSQL Serverless database on `date`
    ```

4.  Zip together the `buildspec.yml` file and the `dbconnect.py` together and upload the zipped file to a `Amazon-QuickSight-PostgreSQL-Automation` folder on your local computer.

5.  Execute these lines on your command line in Cloud9 environment. This will copy the zipped file from GitHub to your s3 bucket created for this project. Once again, ensure that you have the correct s3 bucket name, zipped file name, and directory inserted before running this command.

    ```bash
    git clone https://github.com/osamples/CloudSecurityEngineering.git/Amazon-QuickSight-PostgreSQL-Automation
    cd Amazon-QuickSight-PostgreSQL-Automation
    aws s3 cp codecommit-archive.zip s3://YOUR_S3_BUCKET_NAME
    ```
6.  Go to the CloudFormation console, and create a new stack by uploading a template using the `QuickSight_PostgreSQL_Dataset_CodeBuild_CloudFormation.yaml` file in this repo.

    After your local CloudFormation template loads select **Next**. On the next screen enter a **Stack name** and then make a selection for the Parameters. After you have entered the details select **Next** at the bottom-right of the screen.

    ![CloudFormation](./screenshots/CloudFormation.PNG)

    On the next screen scroll down and choose **Next**. In the following screen scroll down, acknowledge the message (**I acknowledge that AWS CloudFormation might create IAM resources with custom names.**) and choose **Create stack** as shown below.

    ![Deploy Stack](./screenshots/cfn-deploystack.JPG)

7.  Once the CloudFormation stack is created successfully, run the following command in your Cloud9 command line. This will start your first build in CodeBuild.

    ```bash
       aws codebuild start-build --project-name YOUR-CODEBUILD-PROJECT-NAME
    ```

8. Go to the CodeBuild console and verify that the build is complete and was successful.

    ![CodeBuild](./screenshots/CodeBuild.jpg)

9. Go to QuickSight and select `datasets`. Then click `New dataset`. Down at the bottom, you should see a data source that has the name `aurora-postgresql-json-datasource`.  
    
    ![DataSource](./screenshots/DataSource.png)

    You can now follow the console to succesfully create your dataset within QuickSight and begin analyses.

