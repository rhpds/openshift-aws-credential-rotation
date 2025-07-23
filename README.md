# OpenShift CCO Credential Rotation Automation

This Ansible project automates the credential rotation process for OpenShift Container Platform (OCP) clusters using the Cloud Credential Operator (CCO) in `mint` mode on AWS.

The steps outlined in the documentation can be found [here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.12/html/authentication_and_authorization/managing-cloud-provider-credentials#mint-mode-with-removal-or-rotation-of-admin-credential_cco-mode-mint)

## Table of Contents
* [Overview](#overview)
* [Prerequisites](#prerequisites)
  * [1. OpenShift Cluster Requirements](#1-openshift-cluster-requirements)
  * [2. AWS Requirements](#2-aws-requirements)
  * [3. System Requirements](#3-system-requirements)
* [Installation](#installation)
* [IAM Permissions](#iam-permissions)
  * [CCO IAM User Permissions](#cco-iam-user-permissions)
* [Usage](#usage)
  * [Basic Usage](#basic-usage)
  * [Example](#example)
  * [Running Specific Parts](#running-specific-parts)
* [Security Considerations](#security-considerations)
* [Troubleshooting](#troubleshooting)
  * [Common Issues](#common-issues)
  * [Logs](#logs)
* [Project Structure](#project-structure)
* [Contributing](#contributing)
* [License](#license)
* [Support](#support)

## Overview

This automation handles the complete credential rotation workflow:

1. Discovers the current IAM identity by finding which IAM user owns the `aws_access_key_id` in the `aws-creds` secret
2. Deletes the old IAM user entirely (regardless of naming convention)
3. Creates the standardized `ocp-credential-manager-<GUID>` IAM user with proper CCO permissions
4. Generates new AWS access key for the `ocp-credential-manager-<GUID>` IAM user
5. Updates the `aws-creds` secret in the `kube-system` namespace with the new IAM access key
6. Deletes all secret components in OCP froom AWS `credentials_requests` to trigger CCO rotation, verifies that all component secrets are re-created

## Prerequisites

### 1. OpenShift Cluster Requirements
- OpenShift cluster running on AWS
- Cloud Credential Operator (CCO) configured in `mint` mode
- Valid kubeconfig file with cluster-admin permissions

### 2. AWS Requirements
- AWS CLI configured with appropriate profile
- AWS profile with permissions to manage the CCO IAM user and perform credential rotation:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:ListUsers",
                "iam:ListAccessKeys",
                "iam:CreateUser",
                "iam:DeleteUser",
                "iam:CreateAccessKey",
                "iam:DeleteAccessKey",
                "iam:GetUser",
                "iam:AttachUserPolicy",
                "iam:PutUserPolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

**Why these permissions are needed:**
- `iam:ListUsers` - To discover all IAM users and find the one associated with the current access key
- `iam:ListAccessKeys` - To check access keys for each user during discovery
- `iam:CreateUser` - To create the new standardized IAM user
- `iam:DeleteUser` - To delete the old IAM user (which may not follow the naming pattern)
- `iam:CreateAccessKey` - To generate new access keys for the IAM user
- `iam:DeleteAccessKey` - To clean up old access keys
- `iam:GetUser` - To verify user existence and get user details
- `iam:AttachUserPolicy`/`iam:PutUserPolicy` - To attach the CCO policy to the IAM user


### 3. System Requirements
- Python 3.8+
- Ansible 2.15+
- Required Python packages (installed automatically):
  - `kubernetes`
  - `boto3`
  - `botocore`

## Installation

1. Clone this repository:
```bash
git clone <repository-url>
cd creds_rotation_2.0
```

2. Create and activate a Python virtual environment:
```bash
python3 -m venv venv
source venv/bin/activate  # On Linux/Mac
# or
venv\Scripts\activate     # On Windows
```

3. Install Ansible:
```bash
pip install ansible
```

4. Install required Ansible collections:
```bash
ansible-galaxy collection install -r requirements.yml
```

The playbook uses the following collections:
- `amazon.aws` - Official AWS collection for IAM operations
- `kubernetes.core` - Kubernetes/OpenShift cluster interactions

## IAM Permissions

### CCO IAM User Permissions
The `ocp-credential-manager-<GUID>` IAM user will have the following permissions for CCO operations according to the [official documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.12/html/authentication_and_authorization/managing-cloud-provider-credentials#mint-mode-permissions-aws):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateAccessKey",
                "iam:CreateUser",
                "iam:DeleteAccessKey",
                "iam:DeleteUser",
                "iam:DeleteUserPolicy",
                "iam:GetUser",
                "iam:GetUserPolicy",
                "iam:ListAccessKeys",
                "iam:PutUserPolicy",
                "iam:TagUser",
                "iam:SimulatePrincipalPolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

The CCO IAM policy is defined in `group_vars/all.yml` and can be customized if needed:

```yaml
# IAM Policy Configuration
cco_iam_policy_name: "CCOCredentialRotationPolicy"
cco_iam_policy_document:
  Version: "2012-10-17"
  Statement:
    - Effect: "Allow"
      Action:
        - "iam:CreateAccessKey"
        - "iam:CreateUser"
        # ... other permissions
      Resource: "*"
```

To modify the policy:
1. Edit `group_vars/all.yml`
2. Update the `cco_iam_policy_document` section
3. The changes will be applied on the next playbook run

## Usage

### Basic Usage

The playbook can be run with default values or with custom parameters:

```bash
# Using defaults (aws_profile="default", kubeconfig_path="~/.kube/config")
ansible-playbook main.yml

# Or with custom values
ansible-playbook main.yml \
  -e "kubeconfig_path=/path/to/your/kubeconfig" \
  -e "aws_profile=your-aws-profile"
```

### Example

```bash
# Using defaults
ansible-playbook main.yml

# Using custom AWS profile but default kubeconfig location
ansible-playbook main.yml -e "aws_profile=ocp-admin"

# Using custom values for both
ansible-playbook main.yml \
  -e "kubeconfig_path=/home/user/.kube/config" \
  -e "aws_profile=ocp-admin"
```

### Running Specific Parts

Use tags to run only specific parts of the playbook:

```bash
# Only AWS operations
ansible-playbook main.yml -e "..." --tags aws

# Only Kubernetes operations
ansible-playbook main.yml -e "..." --tags kubernetes

# Only cleanup
ansible-playbook main.yml -e "..." --tags cleanup
```

## Security Considerations

- All sensitive credentials are handled securely with `no_log: true`
- Credentials are never stored in plain text files
- Old access keys are automatically deleted after successful rotation
- Use secure storage for kubeconfig files

## Troubleshooting

### Common Issues

1. **"IAM user not found"**
   - Ensure the OpenShift cluster is properly configured with CCO in mint mode
   - Verify the cluster infrastructure name matches the IAM user suffix

2. **"aws-creds secret not found"**
   - Check that CCO is running and properly configured. The creation of this secret is handled by the CCO
   - Verify the cluster is in mint mode (not STS/manual mode)

3. **"Permission denied"**
   - Verify AWS profile has the required IAM permissions
   - Check that kubeconfig has cluster-admin permissions

4. **"Component secrets not recreated"**
   - This operation is handled by the CCO. Monitor CCO logs for errors
   - Check CredentialsRequest resources for issues

### Logs

Check the following logs for troubleshooting:
- CCO logs: `oc logs -n openshift-cloud-credential-operator deployment/cloud-credential-operator`
- Ansible verbose output: Add `-v`, `-vv`, or `-vvv` to the playbook command

## Project Structure

```
creds_rotation_2.0/
├── main.yml                              # Main playbook
├── requirements.yml                      # Ansible collections
├── group_vars/
│   └── all.yml                          # Common variables
├── roles/
│   ├── aws_cco_rotation/
│   │   └── tasks/
│   │       └── main.yml                 # AWS IAM operations
│   └── k8s_secrets/
│       └── tasks/
│           └── main.yml                 # Kubernetes secret management
└── README.md                            # This file
```

## Contributing

1. Follow Ansible best practices
2. Test changes in a non-production environment first
3. Update documentation for any new features
4. Ensure all sensitive operations use `no_log: true`

## License

This project is licensed under the GNU General Public License v2.0 - see below for details.

```
OpenShift Cloud Credential Operator (CCO) AWS Credential Rotation Automation
Copyright (C) 2024

This program is free software; you can redistribute it and/or
modify it under the terms of the GNU General Public License
as published by the Free Software Foundation; either version 2
of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program; if not, write to the Free Software
Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.
```

For the complete license text, visit: https://www.gnu.org/licenses/gpl-2.0.html

## Support

For issues and questions:
- Check the troubleshooting section above
- Review OpenShift CCO documentation
- Consult AWS IAM documentation 
