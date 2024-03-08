# well-architected-framework-automation-workshop
This workshop focuses on demonstrating examples of what is possible with automation, rather than trying to provide comprehensive guidance on how to do automation.

1. Deploy CloudFormation stacks 
	1. [Deploy wafr cloud9 environment](https://static.us-east-1.prod.workshops.aws/e09ebbef-80b5-4b55-94fe-72ad41b22e0d/static/deploy/wafr-cloud9.yml)
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