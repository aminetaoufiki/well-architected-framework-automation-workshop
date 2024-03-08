# well-architected-framework-automation-workshop

This workshop focuses on demonstrating examples of what is possible with automation, rather than trying to provide comprehensive guidance on how to do automation.

## Setting Up the environment

### 1.Deploy CloudFormation stacks

Ensure that you are logged into your account with AdministratorAccess permissions.

```bash
clear ; \
echo "Creating CloudFormation stacks for the workshop..." ; \
aws cloudformation create-stack \
  --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM \
  --template-body "file://wafr-workload.yml" \
  --stack-name "wafr-workload" ; \
aws cloudformation create-stack \
  --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM \
  --template-body "file://wafr-scan-resources.yml" \
  --stack-name "wafr-scan-resources" \
  --parameters "ParameterKey=CallingUserArn,ParameterValue=$(aws sts get-caller-identity --query Arn --output text)" \
               "ParameterKey=WAFRScanRoleName,ParameterValue=WAFRScanRole" \
               "ParameterKey=C9RoleName,ParameterValue=WAFRC9Role" ; \
echo Wait until wafr-scan-resources scanning role has been created... ; \
while [ "$(aws iam list-roles --query 'Roles[?RoleName==`WAFRScanRole`]' | jq -M " .[] | .RoleName" | tr -d \")" != "WAFRScanRole" ] ; do \
  echo "  waiting..." ; \
  sleep 5 ; \
done ; \
aws cloudformation create-stack \
  --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM \
  --template-body "file://wafr-cloud9.yml" \
  --stack-name "wafr-cloud9" \
  --parameters "ParameterKey=CallingUserArn,ParameterValue=$(aws sts get-caller-identity --query Arn --output text)" \
               "ParameterKey=C9SwitchRoleName,ParameterValue=WAFRScanRole" \
               "ParameterKey=C9RoleName,ParameterValue=WAFRC9Role" ; \
echo -e "Stacks created\nIt will take around 15 minutes for stack deployment to complete..."

```

Some parts of this workshop will require an activated AWS SecurityHub. This will incur additional costs for the use of the service (Approx 2 USD per day). Use the following commands to activate SecurityHub (if you want to experiment with this service):

```bash
aws cloudformation create-stack \
  --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM \
  --template-body "file://wafr-sechub-config.yml" \
  --stack-name "wafr-sechub-config" ;\
```

Once Your stacks are deployed, you can delete the deployment scripts

```bash
echo "Removing CloudFormation scripts..." ; \
rm -vf wafr*.yml
```

The "wafr-scan-resources" stack installs a number of resources which are summarised in the Outputs section, including:

- a Cloud9 instance (with pre-provisioned environment)
- an IAM role with restricted/read-only permissions that allows you to run scanning tools without fear of breaking, changing or exposing any resources.
- an S3 Bucket, shared through a Cloud Front distribution, for storing any WAFR scanning reports or other reports gathered during the Well-Architected review preparation/discovery phase
- a CloudFront Link to the S3 bucket

### 2.Set Up the environment

Cloud9 has inbuilt functionality to maintain STS credentials for the user/role that you are logged into the AWS console with. However, as we want to be manually switching roles in the Cloud9 CLI environment, to the WAFR we need to disable this functionality. Disabling managed credentials will ensure that Cloud9 will not inadvertently revert or overwrite any role switching that we do on the command line.

```bash
clear ; \
echo "Disable Cloud9 managed credentials" ; \
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE ; \
echo "Updating the system software" ; \
sudo yum -y update; \
echo "Installing Development Tools..." ; \
sudo yum -y groupinstall "Development Tools" ; \
echo "Installing additional software dependencies..." ; \
sudo yum -y install jq openssl-devel bzip2-devel libffi-devel readline-devel sqlite-devel tk-devel xz-devel ; \
echo "Updating the AWS CLI software" ; \
echo "Removing existing aws-cli..." ; \
sudo yum -y remove aws-cli ; \
rm awscliv2.zip ; \
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" ; \
unzip awscliv2.zip ; \
rm -Rvf ~/environment/awscliv2.zip ~/environment/aws ; \
sudo ./aws/install --update ; \
echo -e '\n#AWS CLI bash-completion\ncomplete -C "/usr/local/bin/aws_completer" aws' >> ~/.bashrc ; \
source ~/.bashrc
echo "Installing Pyenv..." ; \
curl -s https://pyenv.run | bash 1>/dev/null 2>&1 ;\
echo "Adjusting Bash configuration..." ; \
echo -e '\n# Pyenv install \nexport PYENV_ROOT="$HOME/.pyenv" \ncommand -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH" \neval "$(pyenv init -)" \neval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile ; \
# echo -e '\n# Pyenv install\neval "$(pyenv virtualenv-init -)"' >> ~/.bashrc ; \
source ~/.bash_profile ; \
echo "Installing required Python version..." ; \
pyenv install 3.9.16 ; \
pyenv global 3.9.16 ; \
echo "Checking Python version..." ; \
python -m ensurepip --upgrade --user ; \
python -m pip install --upgrade pip

```

Once the dependencies installed, check if the installation went as expected

```bash
clear ; \
echo "Checking Python" ; \
python --version ; \
python -m pip --version ; \
pyenv versions ; \
echo "Checking AWS CLI version" ; \
aws --version ; \
echo "Check that managed credentials are disabled - the assumed role should be \"C9Role\"" ; \
aws sts get-caller-identity --query "Arn" --output text
```

### Switch Roles

You are able to switch roles directly in you CLI environment by generating STS credentials setting specific shell variables.

Run the following commands in the CLI envrionment to setup temporary STS credentials, and store them in a file, and then set the AWS_* variables to switch the role:

```bash
echo "Unsetting any existing variables and reverting to default Cloud9 role" ; \
unset AWS_ACCESS_KEY_ID ; \
unset AWS_SECRET_ACCESS_KEY ; \
unset AWS_SESSION_TOKEN ; \
echo "Setup parameter variables" ; \
export aws_account_id=$(aws sts get-caller-identity --query "Account" --output text) ; \
export role_arn="arn:aws:iam::${aws_account_id}:role/WAFRScanRole" ; \
export role_session_name="Participant" ; \
echo "Set the default region" ; \
aws configure set region $(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text) --profile default ; \
echo "Run the aws cli command to get sts credentials" ; \
aws sts assume-role --role-arn "${role_arn}" \
  --role-session-name "${role_session_name}" \
  | tee ~/wafr-scan-role-tempcredentials.txt ; \
echo "Display the current identity, before switching role:" ; \
aws sts get-caller-identity ; \
echo "Switch role using sts credentials" ; \
export AWS_ACCESS_KEY_ID=$(cat ~/wafr-scan-role-tempcredentials.txt | jq -r .Credentials.AccessKeyId) ; \
export AWS_SECRET_ACCESS_KEY=$(cat ~/wafr-scan-role-tempcredentials.txt | jq -r .Credentials.SecretAccessKey) ; \
export AWS_SESSION_TOKEN=$(cat ~/wafr-scan-role-tempcredentials.txt | jq -r .Credentials.SessionToken) ; \
echo "Display the current identity after switching role, to check that all succeeded" ; \
aws sts get-caller-identity
```

## Well-Architected Workload

In this exercise, we will use the CLI to create a new Well-Architected Workload in the AWS Well-Architected Tool.

```bash
clear ; \
echo "Setup variables to be used in the aws cli command" ; \
export workload_name="wafr-workload" ; \
echo "workload_name = ${workload_name}" ; \
export workload_description="Simulated Workload for automated scans and checks in account ${aws_account_id}, as part of the Well-Architected Automation workshop" ; \
echo "workload_description = ${workload_description}" ; \
export workload_environment="PREPRODUCTION" ; \
echo "workload_environment = ${workload_environment}" ; \
export aws_account_id=$(aws sts get-caller-identity --query "Account" --output text) ; \
echo "aws_account_id = ${aws_account_id}" ; \
export current_arn=$(aws sts get-caller-identity --query "Arn" --output text) ; \
echo "current_arn = ${current_arn}" ; \
export current_region=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text) ; \
echo "current_region = ${current_region}" ; \
echo "Creating new workload in the Well-Architected tool..." ; \
aws wellarchitected create-workload \
  --workload-name "${workload_name}" \
  --description "${workload_description}" \
  --review-owner "${current_arn}" \
  --environment "${workload_environment}" \
  --aws-regions "${current_region}" \
  --account-ids "${aws_account_id}" \
  --lenses "wellarchitected"
```

> [!TIP]
> If you successfully create the resource, the command-line environment will open the response in a "paging" program (like less). You can exit this program by pressing the "q" button.

> [!NOTE]
>If you want to know how to enable Trusted Advisor support manually, please see ([documentation](https://docs.aws.amazon.com/wellarchitected/latest/userguide/activate-ta-for-workload.html))

> [!TIP]
>[AWS Documentation: Define a workload](https://docs.aws.amazon.com/wellarchitected/latest/userguide/define-workload.html)

### Report Bucket Link

In this exercise, we will link the deployed S3 report bucket to new Well-Architected Workload we just created, and create a temporary index.html file that can be viewed using the CloudFront URL.

```bash
clear ; \
echo "Setup variables to be used in the aws cli command" ; \
bucket_name=$(\
  aws cloudformation describe-stacks \
  --stack-name wafr-scan-resources \
  --query 'Stacks[0].Outputs[?OutputKey==`WAFRDataBucket`].OutputValue' \
  --output text) ; \
echo "bucket_name (S3 Report bucket) = ${bucket_name}" ; \
wa_workload_id=$( aws wellarchitected list-workloads \
  --query 'WorkloadSummaries[?WorkloadName==`wafr-workload`].WorkloadId' \
  --output text) ; \
echo "wa_workload_id = ${wa_workload_id}" ; \
cloudfront_hostname=$( \
  aws cloudformation describe-stacks \
  --stack-name wafr-scan-resources \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontURL`].OutputValue' \
  --output text \
  | cut -f3 -d"/") ; \
echo "cloudfront_hostname = ${cloudfront_hostname}" ; \
echo "Update the Well-Architected workload architecture design link with the CloudFront URL" ; \
aws wellarchitected update-workload \
  --workload-id "${wa_workload_id}" \
  --architectural-design \
    "https://${cloudfront_hostname}/${wa_workload_id}/index.html" \
  --output text > /dev/null ; \
echo "Setup of the Architectural Design link in the Well-Architected tool was successful" > _link_setup_successful.html ; \
aws s3 cp _link_setup_successful.html s3://${bucket_name}/${wa_workload_id}/_link_setup_successful.html ; \
rm _link_setup_successful.html ; \
echo "CloudFront URL for WA Workload = https://${cloudfront_hostname}/${wa_workload_id}/index.html"
```

Once the reports bucket URL has been updated in the workload, use the AWS console to view the Well-Architected Tool dashboard . Select the "wafr-workload" from the list of workloads and open the "Properties" section of this Workload. At the bottom right of the "Workload properties" panel, click on the "Architectural design" link. This will open a new tab in your browser - confirm the login to this page, if you are prompted.

### IAM Credential Report

In this exercise, we will use the CLI to create and download a new AWS IAM Credential report, and then upload this to the report bucket.

```bash
clear ; \
echo 'Ensure that you are in the "standard" execution directory' ; \
echo '(ie ~/environment in Cloud9, ~ in the CloudShell)'; \
cd ~/environment 2>/dev/null || cd ~ ; \
echo "Setup variables to be used in the aws cli command" ; \
bucket_name=$(\
  aws cloudformation describe-stacks \
  --stack-name wafr-scan-resources \
  --query 'Stacks[0].Outputs[?OutputKey==`WAFRDataBucket`].OutputValue' \
  --output text) ; \
echo "bucket_name (S3 Report bucket) = ${bucket_name}" ; \
wa_workload_id=$( aws wellarchitected list-workloads \
  --query 'WorkloadSummaries[?WorkloadName==`wafr-workload`].WorkloadId' \
  --output text) ; \
echo "wa_workload_id = ${wa_workload_id}" ; \
cloudfront_url=$( \
  aws cloudformation describe-stacks \
  --stack-name wafr-scan-resources \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontURL`].OutputValue' \
  --output text) ; \
echo "cloudfront_url = ${cloudfront_url}" ; \
echo "Generating IAM credential report..." ; \
aws iam generate-credential-report ; \
aws iam get-credential-report --output text --query Content  | base64 -d > iam_credential_report.csv; \
sleep 5 ; \
echo "Uploading credential report to the S3 reports bucket..." ; \
aws s3 cp iam_credential_report.csv s3://${bucket_name}/${wa_workload_id}/iam_credential_report.csv ; \
echo "Removing Architectural Design link test file from S3 bucket" ; \
aws s3 rm s3://${bucket_name}/${wa_workload_id}/_link_setup_successful.html
```

> [!NOTE]
> It is recommended to generate the IAM credential report before running Steampipe, as documented on Steampipe's [Top 10 Checks page](https://steampipe.io/blog/aws-iam-credential-report-top-10-checks#1-are-you-running-your-credential-report)

## Exercises

### 1.Steampipe Scan

This exercise provides a fast introduction into how to automatically analyse the security state of an AWS account using the open source Security auditing tool Steampipe .

In this exercise, we will use [Steampipe](https://steampipe.io/) to:

- scan the simulated workload account for potential security gaps
- export the scan results into an easily readable HTML file
- export the scan results into JSON, for further processing/parsing

#### 1.1.Install Steampipe

```bash
clear ; \
echo 'Ensure that you are in the "standard" execution directory' ; \
echo '(ie ~/environment in Cloud9, ~ in the CloudShell)'; \
cd ~/environment 2>/dev/null || cd ~ ; \
echo "Installing Steampipe software..." ; \
mkdir -p ~/.local/bin ; \
sudo /bin/sh -c "$(curl -fsSL https://steampipe.io/install/steampipe.sh)" ; \
sudo mv /usr/local/bin/steampipe ~/.local/bin/ ; \
sed -i '/^# Pyenv install.*/i # Adding binary path (for Steampipe)\nPATH=$HOME/.local/bin:$PATH\n' ~/.bashrc ; \
source ~/.bashrc ; \
echo "Installing Steampipe plugins..." ; \
steampipe plugin install steampipe ;\
steampipe plugin install aws ;\
steampipe plugin install awscfn
```

If everything is worked, you should see the scanning tool start scanning...

```shell
Installed 1 mod:

aws_well_architected
└── github.com/turbot/steampipe-mod-aws-compliance@v0.70.0


Running a Steampipe scan and exporting the results...
```

Once completed, you should see a summary of the scan, and a list of report filese generated:

```shell
Summary

OK ........................................................................................................... 111 [===       ]
SKIP .......................................................................................................... 41 [==        ]
INFO ........................................................................................................... 6 [=         ]
ALARM ........................................................................................................ 205 [======    ]
ERROR .......................................................................................................... 7 [=         ]

TOTAL .................................................................................................. 212 / 370 [==========]

File exported to /home/ec2-user/environment/steampipe-mod-aws-well-architected/aws_well_architected..all.20230619T125909.html
File exported to /home/ec2-user/environment/steampipe-mod-aws-well-architected/aws_well_architected..all.20230619T125909.json
File exported to /home/ec2-user/environment/steampipe-mod-aws-well-architected/aws_well_architected..all.20230619T125909.csv
```

Here is an example of what the HTML report will look like [Steampipe Report](./steampipe_report.html)

#### 1.2. Copy reports to S3 report bucket

```bash
clear ; \
echo 'Ensure that you are in the "standard" execution directory' ; \
echo '(ie ~/environment in Cloud9, ~ in the CloudShell)'; \
cd ~/environment 2>/dev/null || cd ~ ; \
cd steampipe-mod-aws-well-architected ;\
echo "Setup variables to be used in the aws cli command" ; \
bucket_name=$(\
  aws cloudformation describe-stacks \
  --stack-name wafr-scan-resources \
  --query 'Stacks[0].Outputs[?OutputKey==`WAFRDataBucket`].OutputValue' \
  --output text) ; \
echo "bucket_name (S3 Report bucket) = ${bucket_name}" ; \
wa_workload_id=$( aws wellarchitected list-workloads \
  --query 'WorkloadSummaries[?WorkloadName==`wafr-workload`].WorkloadId' \
  --output text) ; \
echo "wa_workload_id = ${wa_workload_id}" ; \
for x in aws_well_architected.all.* ; do \
  aws s3 cp ${x} s3://${bucket_name}/${wa_workload_id}/steampipe/${x} ; \
done
```

#### 1.3. Steampipe Dashboards

This exercise provides a fast introduction into the [Steampipe dashboard functionality](https://steampipe.io/docs/reference/mod-resources/dashboard) . In this exercise, we will run the Steampipe tool as a live web-service, within Cloud9, and inspect the functionality it provides.

Different Steampipe modules or "Mods" will provide different functionality and views for your workload. We will work with the AWS Well-Architected, Insights and Thrifty Mods.

#### 1.3.1.Create and inspect the live AWS Well-Architected Mod Steampipe dashboard

In this section, we will setup a live running dashboard, using the [Steampipe AWS Well-Architected mod](https://catalog.workshops.aws/event/dashboard/en-US/workshop/08exercises/02steampipedash#create-and-inspect-the-live-aws-well-architected-mod-steampipe-dashboard:~:text=Steampipe%20AWS%20Well%2DArchitected%20mod%C2%A0) , by running the following commands:

```sh
clear ; \
echo 'Ensure that you are in the "standard" execution directory' ; \
echo ' (ie ~/environment in Cloud9, ~ in the CloudShell)' ; \
cd ~/environment 2>/dev/null || cd ~ ;\
echo "Cloning Steampipe Well-Architected mod into local directory" ; \
git clone https://github.com/turbot/steampipe-mod-aws-well-architected.git ; \
cd steampipe-mod-aws-well-architected ; \
steampipe mod install ; \
echo Starting Steampipe dashboard - type ctrl-c to quit ; \
steampipe dashboard --dashboard-port 8080
```

>[!NOTE]
> You should be able to see and navigate through a Steampipe dashboard

#### 1.3.2.Create and inspect the live AWS Insights Mod Steampipe dashboard

In this section, we will setup a live running dashboard, using the [Steampipe AWS Insights mod](https://catalog.workshops.aws/event/dashboard/en-US/workshop/08exercises/02steampipedash#create-and-inspect-the-live-aws-insights-mod-steampipe-dashboard:~:text=Steampipe%20AWS%20Insights%20mod%C2%A0) , by running the following commands:

```bash
echo 'Ensure that you are in the "standard" execution directory' ; \
echo ' (ie ~/environment in Cloud9, ~ in the CloudShell)' ; \
cd ~/environment 2>/dev/null || cd ~ ; \
echo Cloning Steampipe mod into local directory ; \
git clone https://github.com/turbot/steampipe-mod-aws-insights.git ; \
cd steampipe-mod-aws-insights ; \
echo Starting Steampipe dashboard - type ctrl-c to quit ; \
steampipe dashboard --dashboard-port 8080
```

The dashboard provides views to many different services and resources used in your account.

Try viewing the VPC Detail view to see the way the different VPC resources are connected to each other.

#### 1.3.3.Create and inspect the live AWS Thrifty Mod Steampipe dashboard

In this section, we will setup a live running dashboard, using the [Steampipe AWS Thrifty mod](https://catalog.workshops.aws/event/dashboard/en-US/workshop/08exercises/02steampipedash#create-and-inspect-the-live-aws-insights-mod-steampipe-dashboard:~:text=Steampipe%20AWS%20Thrifty%20mod%C2%A0) , by running the following commands:

```bash
echo 'Ensure that you are in the "standard" execution directory' ; \
echo ' (ie ~/environment in Cloud9, ~ in the CloudShell)' ; \
cd ~/environment 2>/dev/null || cd ~ ;\
echo Cloning Steampipe mod into local directory ; \
git clone https://github.com/turbot/steampipe-mod-aws-thrifty.git ; \
cd steampipe-mod-aws-thrifty ; \
echo Starting Steampipe dashboard - type ctrl-c to quit ; \
steampipe dashboard --dashboard-port 8080
```

> [!TIP]
> Once you have finished viewing the dashboard, you can close the preview window, and exit the dashboard service by interrupting the process (use the standard Linux command ctrl-C).
