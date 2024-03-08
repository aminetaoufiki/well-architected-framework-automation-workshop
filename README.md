# well-architected-framework-automation-workshop
This workshop focuses on demonstrating examples of what is possible with automation, rather than trying to provide comprehensive guidance on how to do automation.

1. Deploy CloudFormation stacks 
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
````
The "wafr-scan-resources" stack installs a number of resources which are summarised in the Outputs section, including:

- a Cloud9 instance (with pre-provisioned environment)
- an IAM role with restricted/read-only permissions that allows you to run scanning tools without fear of breaking, changing or exposing any resources.
- an S3 Bucket, shared through a Cloud Front distribution, for storing any WAFR scanning reports or other reports gathered during the Well-Architected review preparation/discovery phase
- a CloudFront Link to the S3 bucket

2- Set Up your environment

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
