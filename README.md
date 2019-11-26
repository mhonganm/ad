# AzureAD Role Sync Lambda

## COPYRIGHT

Copyright 2019 Amazon Web Services, Inc. or its affiliates. All Rights Reserved.
This file is licensed to you under the AWS Customer Agreement (the "License").
You may not use this file except in compliance with the License.
A copy of the License is located at http://aws.amazon.com/agreement/ .
This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
See the License for the specific language governing permissions and limitations under the License

## Abstract

Many companies choose to master their corporate identities in Microsoft Active Directory, and expose these for Single-Sign-On (SSO) through Azure AD. When using this approach with Amazon Web Services (AWS) there is a need to keep the list of roles in Azure AD in sync with the roles in AWS so that users can be correctly assigned to the roles required for them to perform their duties.

This function automates the synchronization of the AWS IAM SAML roles to the Azure Enterprise AD service. More details on the overarching architecture can bbe found in the Goldmine article (in the links).

The source code associated with this document can be found here:

- [AWS Code Repo for Aad-iam-sync-lambda](https://code.amazon.com/packages/Aad-iam-sync-lambda/trees/mainline)
- [Azure turorial](https://docs.microsoft.com/en-us/azure/active-directory/saas-apps/aws-multi-accounts-tutorial)
- [Azure AD IAM Role Sync Lambda](https://collaborate-corp.amazon.com/nuxeo/nxdoc/default/f290573d-e5fc-4bf8-a620-a197aaedc46d/view_documents)

### FAQs

#### Does Azure not have a native solution and why should I not use this?

Azure best practice it to build 1 Azure AD App for every AWS Account. This [architecture](https://docs.microsoft.com/en-nz/azure/active-directory/saas-apps/amazon-web-service-tutorial) can result in the following problems:

- Its overly cumbersome to implement and automate.
- Azure requires you to create an IAM user for every synchronization job launched which will require IAM system accounts in every AWS account which is currently not best practice.
- Service accounts should have their credentials reset on a regular schedule, this will not scale well in this design.

#### What are the [limits](KnownLimits) of running this function that I need to know about?

- The Azure AD [Manifest](https://docs.microsoft.com/en-us/azure/active-directory/develop/reference-app-manifest) will currently support 1200 application entries this translates to 1199 AWS roles that can be synchronized to a single application.
- AWS Lambda will run for 900 secs (15mins) this should be more than enough for syncing 1200 roles.
- The role names cannot exceed 60 characters, this will break the Manifest value string length.

#### What if I hit these limits, how does it present and how do I scale past this?

- Split into Multiple apps, work with your customer to define the best way to scale the App to Account architecture.
  - You can then deploy new `SAML2` identity provider for each Azure-App, name these distinctly.
  - You can then role a new lambda for each App explicitly updating the `STACK_NAME` and `SAMLIDENTITYPROVIDERNAME` for each deployment.
- Function will stop at 1200, it will give precedence to exiting Azure roles. Functions are listed alphabetically so you may shunt some roles.
- Error message will appear in log files letting you know it is skipping roles.
- The CloudWatch Metric `<ChoosenNameSpace>/<SAMLIdentityProviderName>-skipped` will track the number of Roles found but unable to sync.
- The CloudWatch Metric `<ChoosenNameSpace>/<SAMLIdentityProviderName>-synced` will track the number of Roles found and synchronized.

##### How do I segregated my accounts to support multiple Azure AD Applications

- Accounts can be segregated by AccountId, in a CSV format.
- Accounts can be segregated by Organization OU (in a CSV format).

#### What is it going to cost me to run this function?

- Assuming Free-Tier is not a factor.
- Assuming the function executes to the Maximum synced roles of `1200` and runs every 1/2 hour it should cost 0.15 USD per month.

#### What technologies do I need to know and Support?

1. The Lambda is written in Python,
2. The serverless function is deployed and packaged with Cloudformation,
3. Azure AD,
4. AWS IAM,
5. AWS Organizations.

#### What other architectural concerns are there?

1. This executes in the MasterBilling Account and assumes roles into the children accounts.
   1. The function could be easily abstracted to a `shared-service` account, however for this one-size fits all approach we can only guarantee the existence of the MasterBilling Account.
   2. Microsoft Ad credentials should be rotated, this can be done in the `Secrets Manager` console in Master billing. You can also update the `.env` file and re-run the `make deploy-app`.

## Configuration of Cloud Environments

### Setting up Azure

#### STEP 1: CREATE AN ENTERPRIZE APPLICATION TO REPRESENT YOUR AWS ACCOUNTS

1. Within the Azure Portal, navigate to Azure Active Directory
2. Select Manage "Enterprise applications", and click on "New Application"
3. In the "Add from the gallery" text box enter "AWS", and select the "Amazon Web Services (AWS)" option
   - ![Option](README.md.img/2019-05-08-11-51-03.png)
4. Edit the application name if desired, and click "Add"
5. Navigate to view the new application via: Azure Portal > Azure Active Directory > Enterprise Applications > All Applications > (your application name: e.g. Amazon Web Services (AWS))
6. Select Manage "Single sign-on", and change "Single Sign-on Mode" to "SAML-based Sign-on"
7. (Optional) If you have previously created an Enterprise application from this gallery template then you will need to specify a valid identifier for this application
   - ![identifier](README.md.img/2019-05-08-11-51-59.png)
8. Under "User Attributes": Check the box for "View and edit all other user attributes", then add the following two additional attributes (values are case sensitive):

| Name              | Namespace                                | Source    | Source Attribute         |
| ----------------- | ---------------------------------------- | --------- | ------------------------ |
| `RoleSessionName` | `https://aws.amazon.com/SAML/Attributes` | Attribute | `user.userprincipalname` |
| `Role`            | `https://aws.amazon.com/SAML/Attributes` | Attribute | `user.assignedroles`     |

- ![attribs](README.md.img/2019-05-08-11-52-47.png)
9. Under "SAML Signing Certificate":
   1. Ensure the certificate status is "Active"
   2. Download the "Metadata XML" as we will need this later
   3. If you don't see the download link for `Metadata XML`, click "Save" and then refresh the page
10. Click `Save` to save all changes

![Full Review](README.md.img/2019-07-27-05-40-36.png)

#### STEP 2: DETERMINE THE APPLICATION OBJECT ID

The object id is a random identifier assigned to the Enterprise application by Azure AD.

1. Navigate to Azure Portal > Azure Active Directory > App Registrations > (Your application name e.g. Amazon Web Services (AWS))
2. If you don't see your application on the App Registrations page, ensure "All apps" is selected in the drop down
3. Click on "Manifest"
4. Identify the `objectid` field (usually near the bottom) and note down this value

![Get the objectId](README.md.img/2019-05-08-11-54-42.png)

#### STEP 3: CREATE A NEW USER ACCOUNT WITHIN AZURE AD

This user will be used by the synchronization function when making update calls to Azure AD:

1. Within the Azure Portal, navigate to Azure Active Directory
2. Select Manage "Users", and click on "New User"
3. Fill in the "Name" and "User name" for this user
4. Check the "Show Password" box and note the password for this user
5. Click "Create"
6. In a new browser window, naviate to `https://login.microsoftonline.com` and login as the new user. You will be prompted to change your password
7. **Note** the username and password for this user as these will be required later.
8. **Note** that Azure AD users with the role "User" only have access to resources they "own". At this point, the newly created user has no permissions other than login.
9. Navigate to `Azure Portal > Azure Active Directory > App Registrations > (Your application name e.g. Amazon Web Services (AWS))`.
10. Click on `Settings`, then `Owners` and `Add owner`.
11. Select the user you just created and click `Select`. At this point we are done configuring everything we need within Azure AD.

#### Note: Step the user account to never expire

This will prevent the service from failing every 60 days. A global admin for a Microsoft cloud service can use the Microsoft Azure AD Module for Windows PowerShell to set passwords not to expire for specific users. You can also use Windows PowerShell cmdlets to remove the never-expires configuration or to see which user passwords are set to never expire.

### Testing GraphExplorer interface in Azure

This function will access and update Azure via the Graph Explorer interface, here is how to test the interface using the Azure credentials you have created, this is an important step as it will ensure the user and Graph service has enough access and permission to attach to the API.

#### Step 1: Connect to GraphExplorer

1. Connect to Graph Explorer `https://developer.microsoft.com/graph/graph-explorer`
2. Sign in to the Graph Explorer site using the Global Admin/Co-admin credentials for your tenant.

#### Step 2: Update Graph Explorers permissions to your directory

1. You need to have sufficient permissions to create the roles. Click on modify permissions to get the required permissions.
   - ![GEInterface](README.md.img/2019-07-29-13-55-41.png)
1. Select following permissions from the list (if you don't have these already) and click "Modify Permissions"
   - ![Modify Permissions](README.md.img/2019-07-29-13-57-54.png)
1. This will ask you to login again and accept the consent. After accepting the consent, you are logged into the Graph Explorer again.

#### Step 3: Log in to Graph Explorer as your Sync user

1. Connect to Graph Explorer `https://developer.microsoft.com/graph/graph-explorer`
2. Sign in to the Graph Explorer site using the Sync user you built earlier for your tenant.

#### Step 4: Test the query

1. Change the version dropdown to beta. To fetch all the Service Principals from your tenant, use the following query:
   - `https://graph.microsoft.com/beta/servicePrincipals`
   - ![result](README.md.img/2019-07-29-14-02-56.png)
1. Find the Name of the AWS enterprise application you created.
   - Retrieve the ObjectId
1. Repeat the test again with the ObjectId you created earlier.
   - `https://graph.microsoft.com/beta/servicePrincipals/<objectID>.`
1. Now you will have sufficient privileges to perform the sync.

### Setting up AWS

#### STEP 1: CREATE IDENTITY PROVIDER

The Identity Provider (IDP) represents Azure AD within the AWS environment. The following steps need to be repeated within each child account of your organization to create the IDP in each account.

1. Within the AWS Console, navigate to the IAM Service
2. Click on "Identity providers", and then "Create Provider"
3. Choose `SAML` as the `Provider Type` and enter a recognizable name (e.g. AzureAD). To keep things simple, the same name should be used in each child account
4. For "Metadata document", upload the Manifest XML file you downloaded when creating the AWS Enterprise application within Azure AD
5. Click `Verify` and then "Create"

_Note, if there are multiple child accounts within your organization, it is recommended to automate the above IDP creation via a script using the AWS CLI._

#### STEP 2: CREATE INITIAL ROLE(S)

Now we will create the roles which our AzureAD users will be able to assume/use. These roles will have a trust relationship to the IDP we just created which tells the AWS IAM service to allow AzureAD to authenticate users into these roles.
The following steps need to be repeated within each child account of your organization for each user role you wish to have in that account:

1. Within the AWS Console, navigate to the IAM Service
2. Click on "Roles" and then "Create Role"
3. Under "Select type of trusted entity", click on "SAML 2.0 federation"
4. In the "SAML Provider" drop down, select your newly created IDP
5. Select "Allow programmatic and AWS Management Console access", and click "Next"
   - ![image](README.md.img/2019-05-08-11-58-52.png)
6. Select the Policy (permissions) you wish the new role to have. If this is your first test, it is easiest to select the policy "Administrator Access". The Policies attached to the role can be edited again in future if required.
7. Click "Review"
8. Enter a name for your new Role. It is recommended to prefix the role name to indicate that this role has a trust relationship with AzureAD e.g. AzureAD-AdminRole

### Add Azure Users to the AWS Roles

This part can only be completed once a Sync has occurred at least once, as you will need the roles defined before you can assign user to them.

#### Step 1: Verify the Manifest file has roles in it.

1. Log into Azure as an admin.
2. Navigate to
   1. Azure Active Directory
   2. App Registrations
   3. Amazon Web Services
   4. Manifest

```json
"appRoles": [
  {
    "allowedMemberTypes": [
      "User"
    ],
    "description": "AWS ReadOnly in the Master Billing account2",
    "displayName": "MasterBilling_ReadOnly2",
    "id": "7dfd756e-8c27-4472-b2b7-38c17fc5df5f",
    "isEnabled": true,
    "lang": null,
    "origin": "Application",
    "value": "arn:aws:iam::12345:role/Admin,arn:aws:iam::12345:saml-provider/WAAD"
  },
  {
    "allowedMemberTypes": [
      "User"
    ],
    "description": "AWS ReadOnly in the Master Billing account",
    "displayName": "MasterBilling_ReadOnly",
    "id": "7dfd756e-8c27-4472-b2b7-38c17fc5de5e",
    "isEnabled": true,
    "lang": null,
    "origin": "Application",
    "value": "arn:aws:iam::12345:role/Admin,arn:aws:iam::12345:saml-provider/WAAD"
  }
],
```

#### Step 2: Add a user to the AWS Role

1. 1. Log into Azure as an admin.
2. Navigate to
   1. Azure Active Directory
   2. Enterprise Applications
   3. Amazon Web Services
   4. Users and Groups
   5. Add User
3. Add a user and select the relevant roles.
   - ![Add User](README.md.img/2019-05-08-12-08-01.png)
4. Apply

##### Step 3: Log In with user

1. Go to `https://login.microsoftonline.com/`
1. Select AWS Connector
   - ![AWS Selection](README.md.img/2019-05-08-12-14-50.png)
1. You will be logged into your AWS console.
   - ![SelectConnector](README.md.img/2019-05-08-12-17-53.png)

## Deploying the Function

The code is deployed via the use of Cloudformation and Make.

### Deployment flow

1. Clone the git repository.
1. Install the prerequisites.
1. Setup your IAM credentials.
1. Do a `make test` to confirm your environment.
   1. Do a `make clean` if you have run this before.
1. Deploy to your child account(s).
1. Deploy to your Organizational Master account.
1. Switch IAM credentials.
1. Test

### Prerequisites

In your deployment environment you will require the following tools.

- pip
- aws cli
- make
- git
- IAM credentials

### IAM Credentials for the deployer

In order to deploy this repository you will need an AWS user with an Assume role for every child-account in your organization. These credentials can be specified in your cli environment as follows.

- [Via Environmental Variables](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)
- [Via Named Profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html)
- Via a Named Profile in `AWS_DEFAULT_PROFILE` environmental variable.

I would recommend a named profile for every account you will need access to, the profile will contain the name of the assumed-role you will use to deploy the roles/function in the required account. Then you can change the `AWS_DEFAULT_PROFILE`  to point to the current account you are deploying to. Otherwise you can update the  `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` and `AWS_DEFAULT_REGION`  for every account deployment.

#### Permissions required by deployer

As the deployer you will need permissions to do the following:

- s3
- CloudFormation
- IAM
- Lambda
- CloudWatch Events
- CloudWatch Logs

### IAM Credentials for the `azureSync` lambda function

In order for the AzureSync lambda to assume role into every account it will require a role to be deployed.

#### Permissions required for `azureSync` lambda function

```json
"Statement": [
  {
    "Effect": "Allow",
    "Action": [
      "iam:ListRoles"
    ],
    "Resource": "*"
  }
]
```

The full Cloudformation template for this role exists in the `templates/child-role.json` file.

#### Deployment options for the `child-role.json`

##### Deploy by hand using cloudformation

This could be painful in the long run, and will require you to switch profiles between accounts.

```bash
aws cloudformation deploy \
  --template-file templates/child-role.cfn.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --stack-name ${STACK_NAME} \
    --parameter-overrides \
      MasterAccountId=${MASTERACCOUNTID} \
      AssumeRoleName=${AWS_ASSUME_ROLE_NAME} 
```

##### Use the packaged Makefile to deploy

*This will deploy the Role into the child accounts. Remember you need to have your environment pointing to one of the child accounts (not Masterbilling). You will repeat this step for every applicable child-account.*

```bash
$ make deploy-role

aws sts get-caller-identity
{
    "UserId": "AROAIY2VXLNLYELVAI6DC:botocore-session-1564443537",
    "Account": "916718852657",
    "Arn": "arn:aws:sts::916718852657:assumed-role/OrganizationAccountAccessRole/botocore-session-1564443537"
}
aws cloudformation deploy \
  --template-file templates/child-role.cfn.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --stack-name azure-ad-sync-lambda \
  --parameter-overrides \
    MasterAccountId=${MASTERACCOUNTID} \
    AssumeRoleName=${AWS_ASSUME_ROLE_NAME}

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - azure-ad-sync-lambda
```

*These environmental variables will be the ones defined in your .env file.*

##### Use a LandingZone Template to auto deploy to every new account.

TBA

##### Use Cloudformation StackSets to bulk deploy.

This method is by far the easiest able to dissimilate your role across the entire organization.

*You will have to have a Role in every account by which you can do a a StackSet deploy see [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html).*

###### Create the StackSet

```bash
aws cloudformation create-stack-set --stack-set-name "azure-ad-sync-lambda-childrole" \
      --template-body file://./templates/child-role.cfn.json \
      --description "This is the ChildRole for the azure-ad-sync-lambda function" \
      --execution-role-name WHATEVERSTACKSETROLEYOUHAVEDEFINED \
      --parameters ParameterKey=MasterAccountId,ParameterValue=${MASTERACCOUNTID},UsePreviousValue=false \
                   ParameterKey=AssumeRoleName,ParameterValue=${AWS_ASSUME_ROLE_NAME},UsePreviousValue=false
```

*These environmental variables will be the ones defined in your .env file.*

###### Deploy the StackSet

```bash
aws cloudformation create-stack-instances --stack-set-name "azure-ad-sync-lambda-childrole" \
                                          --regions ap-southeast-2 \
                                          --accounts `aws organizations list-accounts --query "Accounts[*].Id" --output text`
```

*These environmental variables will be the ones defined in your .env file. This command will automatically retrieve ALL the accounts from your organization.*

### Repository Structure

```bash
├── Makefile                      # Makefile for automating the deployment of the function
├── README.md                     # Documentation
├── README.md.img                 # Images and attachments for the document.
├── bootstrap.cfn.sh              # A bootstrap function for folks unable to do a local build.
├── build                         # Part of the bootstrap process.
├── code                          # The lambda function directory.
│   ├── azure-sync.py             # Function
│   ├── requirements.txt          # Code dependencies.
│   └── site-packages             # Where the packages will be installed.
├── deploy  
├── env.template                  # Template for the .env file
├── packaged                      # Where the compiled CloudFormation template will be placed (is ignored in the repo).
└── templates                     # Cloudformation templates
    ├── applicationTemplate.json  # The function template and related resources.
    ├── bootstrap.cfn.json        # The bootstrap template, for folks unable to do a local build.
    └── child-role.cfn.json       # The functions assume-role template that should be distributed to every account you wish to scan.
```

### Environment file

When you deploy this code all the parameters will be loaded from this file, which will be sourced as environmental variables. So when you first try to deploy the project you will need to copy `/env.template` to `.env`  (which is ignored by git). Remember this file does have credentials in it so please shred  and do NOT check into your repository.
1. `AWS_DEFAULT_PROFILE`
   - If you wish to use a aws cli profile to deploy this stack, put it here, if not then remove this line.
2. `STACK_NAME`
   - What is the CloudFormation stack name you will want to use.
   - Must be unique per region per account
   - Default: `azure-ad-sync-lambda`
3. `S3_BUCKET_NAME`
   - The name of a s3 deployment bucket to use for the build.
4. `BOOTSTRAPREPONAME`
   - If you are using the bootstrap template to compile and upload the function. You can specify a transitive repo to use. This will be deleted on bootstrap completion.
   - Default: `AadSyncCodeRepo`
5. `AWS_ASSUME_ROLE_NAME`
   - What name will you give the cross account IAM role.
   - You must keep this consistent across all accounts.
   - Default: azure-ad-sync-role
6. `AZURE_OBJECT_ID`
   - What is the UUID for the Azure Object?
7. `AZURE_TENANT_ID`
   - What is the tenant id of the enterprise application.
   - This will take the form of a domain name ending `onmicrosoft.com` or whichever domain you have registered.
8. `AZURE_USERNAME`
   - What is the username you created for the sync process.
   - Do not use ADMIN users.
9. `AZURE_PASSWORD`
    - What is the password for the above user.
    - ALWAYS enclose the password in `''` to prevent illegal characters.
10. `MASTERACCOUNTID`
    - What is the organizational master account.
    - The code will only deploy the lambda in the organizational master, all other accounts will get an cross-account role deployed.
11. `ACCOUNTLIST`
    - `none` will cause the function to scan every account in the Organizational structure.
    - `123456789012(,)` will cause the function to scan one or more account-ids in a CSV format.
    - `ou-XXXXXX(,)` will cause the function to scan one or more Organizational OUs in a CSV format.
    - If you do not wish to use the Org Structure to find the accounts to review for SAML roles, you can supply a CSV list of account numbers here. If not then leave this as `none`.
12. `SAMLIDENTITYPROVIDERNAME`
    - What is the name of the AWS identity provider you created for this *SingleSignOn* with Azure. You should keep this consistent with all accounts. If you leave this blank we will sync all SAML Federated roles across to the given Azure manifest.
13. `RUNNOTIFICATIONEMAIL`
    - Putting an email address here will setup a distribution alarm and topic for runtime failures of the scanning function.

### Running the Deployment from your local machine

Verify you can meet the pre-requisites first

### `make test`

*This will test the environmental variables and access to AWS.*

```bash
$ make test

aws sts get-caller-identity
{
    "UserId": "AIDAJY4G4BVPBCDQMJNWK",
    "Account": "936415716333",
    "Arn": "arn:aws:iam::936415716333:user/cerberus"
}
```

#### `make clean`

*This will clean up previously deployed code and repos.*

```bash
$ make clean

rm -rf ./code/site-packages || true
rm ./packaged/*.yml || true
rm: ./packaged/*.yml: No such file or directory
mkdir ./code/site-packages

```

#### `make deploy-role` Into Each child account

*This will deploy the Role into the child accounts. Remember you need to have your environment pointing to one of the child accounts (not Masterbilling). You will repeat this step for every applicable child-account.*

```bash
$ make deploy-role

aws sts get-caller-identity
{
    "UserId": "AROAIY2VXLNLYELVAI6DC:botocore-session-1564443537",
    "Account": "916718852657",
    "Arn": "arn:aws:sts::916718852657:assumed-role/OrganizationAccountAccessRole/botocore-session-1564443537"
}
aws cloudformation deploy \
		--template-file templates/child-role.cfn.json \
		--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
		--stack-name azure-ad-sync-lambda \
        --parameter-overrides \
					MasterAccountId=936415716333 \
        	AssumeRoleName=azure-ad-sync-role

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - azure-ad-sync-lambda
```

#### `make deploy-app` Into the MasterBilling Account

*This will download the dependencies, package up the code and deploy via cloudformation and the CLi*.

```bash
$ make deploy

pip install -r code/requirements.txt --target ./code/site-packages
DEPRECATION: Python 2.7 will reach the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 won't be maintained after that date. A future version of pip will drop support for Python 2.7.
Collecting azure-graphrbac==0.53.0 (from -r code/requirements.txt (line 1))
  Using cached https://files.pythonhosted.org/packages/da/a8/3d3d6fe8458b2b07bad10195c79928ea9ba87b5cc0c08903b387dd27c6f0/azure_graphrbac-0.53.0-py2.py3-none-any.whl
Collecting boto3==1.9.88 (from -r code/requirements.txt (line 2))
 
 ...

 requests-oauthlib-1.2.0 s3transfer-0.1.13 six-1.12.0 typing-3.7.4 urllib3-1.25.3

aws cloudformation package \
		--template-file templates/applicationTemplate.json \
		--output-template-file packaged/sam.out.cfn.yml \
		--s3-bucket cerberus-deploy-bucket \
		--s3-prefix cfn
Uploading to cfn/f6b74fe0fd91dd4d4bf1ef5cfb074567  13036706 / 13036706.0  (100.00%)
Successfully packaged artifacts and wrote output template to file packaged/sam.out.cfn.yml.

Execute the following command to deploy the packaged template
aws cloudformation deploy --template-file /Users/sidsig/eDevel/com/amazon/aws/Aad-iam-sync-lambda/packaged/sam.out.cfn.yml --stack-name <YOUR STACK NAME>
aws cloudformation deploy \
		--template-file packaged/sam.out.cfn.yml \
		--capabilities CAPABILITY_IAM \
		--stack-name azure-ad-sync-lambda \
        --parameter-overrides \
					AccountList=none \
					MasterAccountId=936415716333 \
					BackupRetensionInDays=30 \
        	AssumeRoleName=azure-ad-sync-role \
					AwsAMLProviderName= \
					AzureObjectId=bf55b9b2-da79-4ecb-8361-12345678 \
					AzureTenant=blah.onmicrosoft.com \
					AzureUserName=blah@blah.onmicrosoft.com \
					AzurePassword=blahblahblah

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - azure-ad-sync-lambda
```

### Running the Deployment bootstrap from CodeBuild

If you cannot deploy the repository locally from your machine you can opt to use a self-terminating deployment pipeline

#### Deployment workflow for `bootstrap.cfn.sh`

This shell script will launch the bootstrap pipeline to deliver the code into your account. It still requires you to update a valid `.env` file.

1. Source the `.env` file an export the environment variables.
2. Verify the AWS credentials.
3. Zip up the entire package.
4. Upload to an s3 bucket.
5. Remove any failed deployments of the bootstrap.
6. Launch the CFN template
   1. It will build a CodeCommit repo from the uploaded zip file.
   2. It will build a CodePipeline to do the deployment
   3. It will lint the code and ensure it is sound.
   4. It will package and deploy the code using cloudformation.
   5. It will self-terminate all resources that were deployed to bootstrap the function.
7. It will unset all the environmental variables set.

#### If you do not have a linux environnement?

You can also deliver the bootstrap via the AWS console via the following process.

1. Review the `./env.template` file and ensure you have all this information to hand.
2. Using the console create a Git repository with this repo.
3. Use the cloudformation console to launch the `./template/bootstrasp.cfn.json` file.
4. Fill in all the `env.template` variables into the cfn-parameters.

## Monitoring

### Dashboard
The solution comes with a comprehensive set of metrics, gathered in a CloudWatch Dashboard for easy access and monitoring. A new dashboard will exist for every function your deploy.

![CloudWatch Dashboard](README.md.img/2019-11-11-12-05-03.png)

#### Account Numbers

This will display metrics for Total accounts found as well as accounts successfully scanned.

#### Roles Synchronized

This will display the role limit, roles synchronized and roles skipped counts.

#### Azure AdSyncProcess

This will record function runs and whether they were successfully or failed. This can also send SNS email notification on failure.

#### AzureRoles Synchronized

This is a counter for the total currently synchronized roles.

#### Critical issues Log Insights

This is a running log of Critical issues encountered.

### Standard Lambda monitoring

CloudWatch has many gathered metrics to support the running of the function.  This is accessible from the Lambda Console.

![CloudWatch](README.md.img/2019-07-31-11-36-33.png)

If the Synchronization function is to be used on an on-going basis within as part of your AWS Environment.

## Testing

Once deployed successfully you can run the lambda as follows in the `Masterbilling` account.

### Find the function name

List out the Lambda functions int he master account and look for the function name you just deployed.

```bash
$ aws lambda list-functions --query "Functions[*].FunctionName" --output text

azure-ad-sync-lambda-AzureADSyncFunc-1PMJR2EEXPNZS

```

### Execute the function

```bash
aws lambda invoke  --function-name azure-ad-sync-lambda-AzureADSyncFunc-1PMJR2EEXPNZS /tmp/output.txt
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

### Find the StreamName for the function

```bash
$ aws logs describe-log-streams --log-group-name /aws/lambda/azure-ad-sync-lambda-AzureADSyncFunc-1PMJR2EEXPNZS --descending --query 'logStreams[*].[logStreamName, creationTime]' --output text | head -1

2019/07/30/[$LATEST]fc6a3ef7bdea4598bd09c9b8c43b8740	1564478926532
```

- `--log-group-name` 
  - will be `/aws/lambda/` plus the function name retrieved above. This will return the latest record.

### Extract the Running logs

This will return the logs stored by the function when it executed.

```bash
$ aws logs get-log-events --log-group-name /aws/lambda/azure-ad-sync-lambda-AzureADSyncFunc-1PMJR2EEXPNZS --log-stream-name '2019/07/30/[$LATEST]fc6a3ef7bdea4598bd09c9b8c43b8740' --query 'events[*].[timestamp,message]' --output text


1564478927491	[WARNING]	2019-07-30T09:28:47.491Z		Running in lambda aws mode.
1564478927510	[INFO]	2019-07-30T09:28:47.510Z		Found credentials in environment variables.
1564478928386	[INFO]	2019-07-30T09:28:48.386Z		Successfully authenticated to account.
1564478928387	START RequestId: 2f075a32-0d33-4891-9f7a-a5ccf370c13d Version: $LATEST
1564478928387	[WARNING]	2019-07-30T09:28:48.387Z	2f075a32-0d33-4891-9f7a-a5ccf370c13d	>>>>>>> BEGIN the Azure-sync function.
1564478928387	[INFO]	2019-07-30T09:28:48.387Z	2f075a32-0d33-4891-9f7a-a5ccf370c13d	Initializing the AppRole Object

...

1564478948638	[WARNING]	2019-07-30T09:29:08.638Z	2f075a32-0d33-4891-9f7a-a5ccf370c13d	Manifest has NOT changed so will skip update.
1564478948639	END RequestId: 2f075a32-0d33-4891-9f7a-a5ccf370c13d
1564478948639	REPORT RequestId: 2f075a32-0d33-4891-9f7a-a5ccf370c13d	Duration: 20251.82 ms	Billed Duration: 20300 ms 	Memory Size: 128 MB	Max Memory Used: 111 MB
```

- `--log-group-name`
  - will be `/aws/lambda/` plus the function name retrieved above. This will return the latest record.
- `--log-stream-name`
  - Will be garnered from the previous query and MUST be quoted.

### Review and Possible Errors

| Message in logs                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Reasoning                                                       | Outcome                                                                     |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `Manifest has NOT changed so will skip update.`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Indicates no changes have occurred between Azure and AWS        | Ensure there are changes to push.                                           |
| `Manifest has changed so will backup and update.`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Indicates a change was detected and the manifest updated.       |                                                                             |
| `Uploaded file [2019/07/30/23/32/26/beforeManifestFilebf55b9b2-da79-4ecb-8361-a371638f5ba4.json] to bucket [azure-ad-sync-lambda-manifestbackupbucket-1q4161afgaldq]`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | The before snapshot of the file and the path in the bucket.     |                                                                             |
| `2019/07/30/23/32/26/afterManifestFilebf55b9b2-da79-4ecb-8361-a371638f5ba4.json] to bucket [azure-ad-sync-lambda-manifestbackupbucket-1q4161afgaldq`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | The after snapshot of the file and the path in the bucket.      |                                                                             |
| `Error authenticating to Azure with user [testy@azureevilsid.onmicrosoft.com], error was [, AdalError: Get Token request returned http error: 401 and server response: {\"error\":\"user_password_expired\",\"error_description\":\"AADSTS50055: Password is expired.\\r\\nTrace ID: 127537a8-344b-42f1-abad-7d7ac40e0200\\r\\nCorrelation ID: 70ccd8a4-cf1d-461a-99b7-2b91c0aefd5f\\r\\nTimestamp: 2019-08-02 23:21:08Z\",\"error_codes\":[50055],\"timestamp\":\"2019-08-02 23:21:08Z\",\"trace_id\":\"127537a8-344b-42f1-abad-7d7ac40e0200\",\"correlation_id\":\"70ccd8a4-cf1d-461a-99b7-2b91c0aefd5f\",\"suberror\":\"user_password_expired\",\"password_change_url\":\"https://portal.microsoftonline.com/ChangePassword.aspx\"}]` | Password for user has expired.                                  | Need to make the sync user non-expiry                                       |
| `[WARNING]	2019-08-03T23:40:51.621Z	4adb61f5-c7c4-4dbb-89d6-36a2a600e75a	Unable to assume role azure-ad-sync-role in account 936415716333, with the error [An error occurred (AccessDenied) when calling the AssumeRole operation: Access denied]`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | You have not deployed the child-account-role into this account. | See deployment steps and `make deploy-role` step.                           |
| `[CRITICAL]	2019-08-04T00:28:45.186Z	ebcb7d28-15f0-4ae8-bd32-e906dd480a51	Error authenticating to Azure with user [testy@azureevilsid.onmicrosoft.com]`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Your password and/or azure username is incorrect.               | Update the details in `Secrets-Manager` or in the `.env` file and redeploy. |

## <a name="KnownLimits">KnownLimits</a>

### Azure Manifest File

- AppRoleValue Length
  - `Failed to update Amazon Web Services (AWS) application. Error detail: Entitlement ClaimValue too long`
  - The maximum limit to this is 121 characters.
- AppRoles you can add
  - `Failed to update application xxxxxx. Error details: The size of the manifest has exceeded its limit. Please reduce the number of values and retry your request`
  - Limit is 1200 entries.
- AWS Roles soft-limit
  - You can add a limit of 1000 roles per AWS account.
  - This can be extended. through a service request.
  
### Stress and Volume

| Number of AWS Roles | Lambda time to sync                                            |
| ------------------- | -------------------------------------------------------------- |
| 24                  | Duration: 24115 ms Memory Size: 128 MB Max Memory Used: 117 MB |
| 504                 | Duration: 39100 ms Memory Size: 128 MB Max Memory Used: 118 MB |
| 1000                | Duration: 72100 ms Memory Size: 128 MB Max Memory Used: 122 MB |
| 1200                | Duration: 74600 ms Memory Size: 128 MB Max Memory Used: 121 MB |

## Sample Data

### Manifest File example

```json
{
	"id": "bf55b9b2-da79-4ecb-8361-a371123456",
	"acceptMappedClaims": null,
	"accessTokenAcceptedVersion": null,
	"addIns": [],
	"allowPublicClient": false,
	"appId": "3d06bf80-46e1-4ee5-ad62-123456",
	"appRoles": [
		{
			"allowedMemberTypes": [
				"User"
			],
			"description": "ASSUREROACCESS",
			"displayName": "[123456789012] ASSUREROACCESS",
			"id": "9fe37d57-3d41-552d-9da7-99123456",
			"isEnabled": true,
			"lang": null,
			"origin": "Application",
			"value": "arn:aws:iam::123456789012:role/ASSUREROACCESS,arn:aws:iam::123456789012:saml-provider/ASSSURE"
		},
		{
			"allowedMemberTypes": [
				"User"
			],
			"description": "msiam_access",
			"displayName": "msiam_access",
			"id": "7dfd756e-8c27-4472-b2b7-12345678",
			"isEnabled": true,
			"lang": null,
			"origin": "Application",
			"value": null
		},
		{
			"allowedMemberTypes": [
				"User"
			],
			"description": "ASSSURE-RO-SAML",
			"displayName": "[123456789012] ASSSURE-RO-SAML",
			"id": "0370e0c9-2cb7-5ecd-8fe6-f1f1cc09a324",
			"isEnabled": true,
			"lang": null,
			"origin": "Application",
			"value": "arn:aws:iam::123456789012:role/ASSSURE-RO-SAML,arn:aws:iam::123456789012:saml-provider/ASSSURE"
		}
	],
	"oauth2AllowUrlPathMatching": false,
	"createdDateTime": "2019-06-21T21:11:45Z",
	"groupMembershipClaims": null,
	"identifierUris": [
		"https://signin.aws.amazon.com/saml"
	],
	"informationalUrls": {
		"termsOfService": null,
		"support": null,
		"privacy": null,
		"marketing": null
	},
	"keyCredentials": [],
	"knownClientApplications": [],
	"logoUrl": null,
	"logoutUrl": null,
	"name": "Amazon Web Services (AWS)",
	"oauth2AllowIdTokenImplicitFlow": true,
	"oauth2AllowImplicitFlow": false,
	"oauth2Permissions": [
		{
			"adminConsentDescription": "Allow the application to access Amazon Web Services (AWS) on behalf of the signed-in user.",
			"adminConsentDisplayName": "Access Amazon Web Services (AWS)",
			"id": "19bb96cb-095e-4a83-8e10-813c04c07eb9",
			"isEnabled": true,
			"lang": null,
			"origin": "Application",
			"type": "User",
			"userConsentDescription": "Allow the application to access Amazon Web Services (AWS) on your behalf.",
			"userConsentDisplayName": "Access Amazon Web Services (AWS)",
			"value": "user_impersonation"
		}
	],
	"oauth2RequirePostResponse": false,
	"optionalClaims": null,
	"orgRestrictions": [],
	"parentalControlSettings": {
		"countriesBlockedForMinors": [],
		"legalAgeGroupRule": "Allow"
	},
	"passwordCredentials": [],
	"preAuthorizedApplications": [],
	"publisherDomain": "azureevilsid.onmicrosoft.com",
	"replyUrlsWithType": [
		{
			"url": "https://signin.aws.amazon.com/saml",
			"type": "Web"
		}
	],
	"requiredResourceAccess": [],
	"samlMetadataUrl": null,
	"signInUrl": "https://signin.aws.amazon.com/saml?metadata=aws|ISV9.1|primary|z",
	"signInAudience": "AzureADMyOrg",
	"tags": [],
	"tokenEncryptionKeyId": null
}
```
