# AWS Zero Trust Architecture for Service-to-Service Workloads

## Overview

This project demonstrates how to implement **Zero Trust principles** for service-to-service architectures on AWS. It walks through a realistic microservices scenario where a calling service (ServiceA) communicates with a target service (ServiceB) through AWS PrivateLink, and progressively hardens the security posture using layered, identity-centric controls.

The lab moves away from legacy network-centric access controls and toward a modern Zero Trust model by leveraging **IAM authorization**, **API Gateway resource policies**, **VPC endpoint policies**, and **security group rules** in combination.

---

## Architecture

The diagram below illustrates the full architecture used in this project:

![Architecture Diagram](./architecture-diagram.png)

### Components

**Calling Services (ServiceA VPC)**
- **Expected Caller** — An EC2 instance in Private Subnet 1 (`10.199.0.0/24`) that is authorized to call ServiceB via the ServiceA VPC Endpoint.
- **Unwanted Callers #1, #2, #3** — Lambda functions within the ServiceA VPC that should be blocked from reaching ServiceB.
- **Unwanted Caller #4** — An EC2 instance in a separate **Other VPC** that attempts to reach ServiceB through a different VPC endpoint.
- **ServiceA Endpoint** — An API Gateway VPC Endpoint (`ServiceA-APIGW-EP`) used by callers in the ServiceA VPC.
- **Other Endpoint** — A different VPC endpoint used by Unwanted Caller #4 from the Other VPC.

**Target Services**
- **Order History Service (ServiceB)** — Consists of a `ServiceB API` (API Gateway), a `ServiceB Backend Lambda`, and an `Order Table` (DynamoDB).
- **Unknown API Service** — A separate private API that the Expected Caller should NOT be able to reach.

**Connectivity**
- All traffic from calling services reaches the target services through **AWS PrivateLink**.

---

## Callers Reference Table

| Caller Name | Caller Type | Target API | Call Path | Caller CIDR | SigV4 Support | IAM Permissions | Desired Outcome |
|---|---|---|---|---|---|---|---|
| Expected Caller | EC2 Instance | ServiceB | Via ServiceA Endpoint | `10.199.0.0/24` | ✅ Yes | ✅ Yes | ✅ Allow |
| Expected Caller | EC2 Instance | Unknown API | Via ServiceA Endpoint | `10.199.0.0/24` | ✅ Yes | ✅ Yes | ❌ Block |
| Unwanted Caller #1 | Lambda | ServiceB | Via ServiceA Endpoint | `10.199.1.0/24` | ❌ No | ❌ No | ❌ Block |
| Unwanted Caller #2 | Lambda | ServiceB | Via ServiceA Endpoint | `10.199.0.0/24` | ❌ No | ❌ No | ❌ Block |
| Unwanted Caller #3 | Lambda | ServiceB | Via ServiceA Endpoint | `10.199.0.0/24` | ✅ Yes | ❌ No | ❌ Block |
| Unwanted Caller #4 | EC2 Instance | ServiceB | Via Other Endpoint | `10.199.0.0/24` | ✅ Yes | ✅ Yes | ❌ Block |

---

## Prerequisites

Familiarity with the following AWS services is required before working through this project:

- Amazon API Gateway
- AWS Identity and Access Management (IAM)
- Amazon Virtual Private Cloud (Amazon VPC)
- VPC Endpoints (Interface Endpoints)

---

## Objectives

1. Review the current state of the service-to-service architecture.
2. Review the existing security controls applied in the solution.
3. Run an assessment to evaluate the current security posture.
4. Improve the security posture using **IAM authorization** on Amazon API Gateway.
5. Improve the security posture using an **API Gateway resource policy**.
6. Improve the security posture using an **Amazon VPC Endpoint policy**.
7. Improve the security posture by tuning **VPC endpoint security group rules**.

---

## Security Controls Overview

Security controls can be applied at multiple layers in this architecture:

**API Gateway**
- **IAM Authorization** — Requires callers to have valid IAM credentials and sign requests using SigV4. When set to `NONE`, the API accepts calls from any entity.
- **Resource Policy** — An IAM resource policy attached to the API that enforces identity-centric and network-centric conditions.
- **Endpoint Type** — Both `ServiceB API` and `Unknown API` are set to `Private`, meaning they can only be called through an API Gateway VPC Endpoint from within a VPC.

**VPC Endpoint**
- **Resource Policy** — Restricts which IAM principals can invoke which API resources through this endpoint, enforcing identity rules at the network boundary.
- **Security Group** — Defines inbound/outbound rules to control traffic traversing the VPC endpoint at the network layer.

---

## Step-by-Step Tasks

### Task 1 — Review Existing Security Controls

**API Gateway**

1. In the AWS Console, navigate to **API Gateway**.
2. Select **ServiceBAPI** → `/orders` → **GET** → **Method Request** tab.
3. Verify that **Authorization** is set to `NONE`.
   > When Authorization is `NONE`, the API accepts calls from sources without valid IAM credentials and without SigV4 signing.
4. In the left panel, select **Resource policy** and review the current policy.
   > The existing policy allows any call originating from the `10.199.0.0/24` IP range — this overlaps with both Private Subnet 1 in the ServiceA VPC and the Other VPC, creating an overly permissive rule.

**VPC Endpoint**

1. Navigate to **VPC** → **Endpoints**.
2. Locate the API Gateway VPC endpoint in the ServiceA VPC subnets.
3. Tag the endpoint for easy identification:
   - **Key:** `Name`
   - **Value:** `ServiceA-APIGW-EP`
4. Review the **Policy** tab — the current policy allows all traffic to any Private API in the region with no restrictions.

> **Observation:** The current architecture relies almost entirely on network-centric controls, which conflicts with Zero Trust principles. Most unwanted callers can currently reach the ServiceB API.

---

### Task 2 — Assess the Current Security Posture

Connect to `ServiceA-Instance` via **AWS Systems Manager Session Manager** and run the assessment script:

```bash
python3 scanner.py
```

#### `scanner.py`

```python
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: MIT-0

import json
import time
import os
import boto3
import botocore
from dotenv import load_dotenv

import service_a_caller
import service_a_unknownapi

load_dotenv()

region = os.environ["api_region"]
boto3_session = boto3.session.Session(region_name=region)
ssm_client = boto3_session.client('ssm')

GREEN = '\033[92m'
WARNING = '\033[93m'
OTHER = '\033[95m'
FAIL = '\033[91m'
ENDC = '\033[0m'
BOLD = '\033[1m'
CROSS = '\u2718'
CHECK = '\u2714'

def get_ssm_cmd(instance_id):
    try:
        cmd1 = "python3 /tmp/lab/service_a_caller_sigv4.py"
        response = ssm_client.send_command(
            InstanceIds=[instance_id],
            DocumentName='AWS-RunShellScript',
            Parameters={"commands": [cmd1]}
        )
    except botocore.exceptions.ClientError as error:
        if error.response['Error']['Code'] == 'AccessDeniedException':
            print('There is a temporary issue. Please wait one minute then try again.')
            quit()
        else:
            raise error

    command_id = response.get('Command', {}).get("CommandId", None)
    timeout = 30
    start_time = time.time()
    while True:
        if time.time() - start_time > timeout:
            return "ConnectTimeout: Command execution timed out after 30 seconds"
        response = ssm_client.list_command_invocations(CommandId=command_id, Details=True)
        if len(response['CommandInvocations']) == 0:
            time.sleep(0.5)
            continue
        invocation = response['CommandInvocations'][0]
        if invocation['Status'] not in ('Pending', 'InProgress', 'Cancelling'):
            break
        time.sleep(0.5)
    command_plugin = invocation['CommandPlugins'][-1]
    return command_plugin['Output']

def get_response(caller):
    lambda_client = boto3_session.client('lambda')
    response = lambda_client.invoke(FunctionName=caller)
    return json.loads(response['Payload'].read())

def parse_result(response):
    response = str(response)
    if "SUCCESS" in response:
        return ("Allowed", "-")
    elif "not authorized" in response:
        if "hit-apigw" in response:
            return ("Blocked", "API Gateway")
        else:
            return ("Blocked", "VPC endpoint")
    elif "Missing Authentication Token" in response:
        return ("Blocked", "API Gateway")
    elif "ConnectTimeout" in response:
        return ("Blocked", "Security Group")
    else:
        return (response[:100], "unknown")

def print_results(callers):
    titles = ['Check', 'Result', 'Enforced@', 'Desired outcome']
    longest_string = 24
    line = '   '.join(str(x).ljust(longest_string + 4) for x in titles)
    print(BOLD + line + ENDC)
    print('-' * len(line))

    for i, caller in enumerate(callers):
        if caller[0] == "service_a_caller":
            check_label = "Expected Caller"
            response = service_a_caller.main()
        elif caller[0] == "service_a_unknownapi":
            check_label = "Expected Caller-Unknown API"
            response = service_a_unknownapi.main()
        elif caller[0][:3] != "arn":
            check_label = f'Unwanted Caller #{i-1}'
            response = get_ssm_cmd(caller[0])
        else:
            check_label = f'Unwanted Caller #{i-1}'
            response = get_response(caller[0])

        result = parse_result(response)
        row = [check_label, result[0], result[1]]
        line = '   '.join(str(x).ljust(longest_string + 4) for x in row)

        if row[1] == "Allowed" and caller[1] == "wanted":
            print(GREEN + line + ' ' * 10 + CHECK + ENDC)
        elif row[1] == "Allowed" and caller[1] == "unwanted":
            print(FAIL + line + ' ' * 10 + CROSS + ENDC)
        elif row[1] == "Blocked" and caller[1] == "wanted":
            print(FAIL + line + ' ' * 10 + CROSS + ENDC)
        elif row[1] == "Blocked" and caller[1] == "unwanted":
            print(GREEN + line + ' ' * 10 + CHECK + ENDC)

def main():
    unwanted_callers_str = ssm_client.get_parameter(
        Name=os.environ["unwanted_callers_parameter"]
    )['Parameter']['Value']
    unwanted_callers = unwanted_callers_str.split(",")

    all_callers = [("service_a_caller", "wanted")]
    all_callers.extend([("service_a_unknownapi", "unwanted")])
    all_callers.extend([(c, "unwanted") for c in unwanted_callers])

    print("\n> Started scanning ...\n")
    print_results(all_callers)
    print("\n> Finished scanning.\n")

if __name__ == "__main__":
    main()
```

#### Baseline Assessment Output

```
> Started scanning ...

Check                           Result               Enforced@             Desired outcome
------------------------------------------------------------------------------------------------
Expected Caller                 Allowed              -                            ✔
Expected Caller-Unknown API     Allowed              -                            ✘
Unwanted Caller #1              Blocked              API Gateway                  ✔
Unwanted Caller #2              Allowed              -                            ✘
Unwanted Caller #3              Allowed              -                            ✘
Unwanted Caller #4              Allowed              -                            ✘

> Finished scanning.
```

#### Baseline Assessment Analysis

| Caller | Target | Result | Reason |
|---|---|---|---|
| Expected Caller | ServiceB | ✅ Allowed | Resource policy allows `10.199.0.0/24`; VPC Endpoint provides the path |
| Expected Caller | Unknown API | ❌ Allowed (should be blocked) | Same permissive resource policy applies |
| Unwanted Caller #1 | ServiceB | ✅ Blocked | Source IP is `10.199.1.0/24`, outside the allowed CIDR |
| Unwanted Caller #2 | ServiceB | ❌ Allowed (should be blocked) | Source IP falls within the allowed CIDR |
| Unwanted Caller #3 | ServiceB | ❌ Allowed (should be blocked) | Source IP falls within the allowed CIDR |
| Unwanted Caller #4 | ServiceB | ❌ Allowed (should be blocked) | Source IP `10.199.0.0/24` overlaps with Other VPC subnet |

> Only **2 of 6** API call patterns meet the desired security outcome. This is a critical gap that needs to be addressed.

---

### Task 3 — Enable IAM Authorization on API Gateway

Enable IAM-based authorization to require all API callers to sign their requests using **SigV4**.

1. Navigate to **API Gateway** → **ServiceBAPI** → `/orders` → **GET** → **Method Request** → **Edit**.
2. Set **Authorization** to **AWS IAM** and choose **Save**.
3. Choose **Deploy API** → Stage: `api` → **Deploy**.

Update the ServiceA caller script (`service_a_caller_sigv4.py`) to sign requests with SigV4:

```python
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: MIT-0

from dotenv import load_dotenv
import os
import boto3
import requests
from aws_requests_auth.boto_utils import BotoAWSRequestsAuth

load_dotenv()
region = os.environ["api_region"]

def call_api(api_id: str, api_key=None):
    host = api_id + '.execute-api.' + region + '.amazonaws.com'
    base_url = f'https://{host}/api'
    get_url = f'{base_url}/{os.environ["api_resource"]}'

    auth = BotoAWSRequestsAuth(aws_host=host, aws_region=region, aws_service='execute-api')
    try:
        response = requests.get(get_url, headers={'x-api-key': api_key}, timeout=15, auth=auth)
    except requests.exceptions.RequestException as e:
        raise SystemExit(e)
    return response

def main():
    boto3_session = boto3.session.Session(region_name=region)
    client = boto3_session.client('ssm')
    api_id = client.get_parameter(Name=os.environ["api_id_parameter"])['Parameter']['Value']
    api_secret_arn = client.get_parameter(Name=os.environ["api_secret_parameter"])['Parameter']['Value']

    client = boto3_session.client('secretsmanager')
    api_key = client.get_secret_value(SecretId=api_secret_arn)["SecretString"]

    response = call_api(api_id, api_key)
    return response.text

if __name__ == "__main__":
    print(main())
```

Run the assessment again:

```bash
python3 scanner.py
```

#### Assessment Output (After IAM Authorization)

```
> Started scanning ...

Check                           Result               Enforced@             Desired outcome
------------------------------------------------------------------------------------------------
Expected Caller                 Allowed              -                            ✔
Expected Caller-Unknown API     Allowed              -                            ✘
Unwanted Caller #1              Blocked              API Gateway                  ✔
Unwanted Caller #2              Blocked              API Gateway                  ✔
Unwanted Caller #3              Allowed              -                            ✘
Unwanted Caller #4              Allowed              -                            ✘

> Finished scanning.
```

> **Improvement:** Unwanted Caller #2 is now blocked because it does not support SigV4. Unwanted Callers #3 and #4 remain unblocked.

---

### Task 4 — Improve the API Gateway Resource Policy

Update the resource policy of `ServiceBAPI` to enforce both identity-centric and network-centric conditions using `aws:PrincipalAccount` and `aws:SourceVpce`.

1. Navigate to **API Gateway** → **ServiceBAPI** → **Resource policy** → **Edit**.
2. Delete the existing policy and replace it with the following, substituting the placeholder values:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:*:*:*"
        },
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:*:*:*",
            "Condition": {
                "StringNotEquals": {
                    "aws:PrincipalAccount": "INSERT_AWS_ACCOUNT_ID"
                }
            }
        },
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:*:*:*",
            "Condition": {
                "StringNotEquals": {
                    "aws:SourceVpce": "INSERT_APIGW_VPC_ENDPOINT_ID"
                }
            }
        }
    ]
}
```

| Placeholder | Description |
|---|---|
| `INSERT_AWS_ACCOUNT_ID` | The AWS Account ID of the ServiceA account |
| `INSERT_APIGW_VPC_ENDPOINT_ID` | The VPC Endpoint ID of `ServiceA-APIGW-EP` |

3. Choose **Save changes** → **Resources** → **Deploy API** → Stage: `api` → **Deploy**.

#### Assessment Output (After Resource Policy Update)

```
> Started scanning ...

Check                           Result               Enforced@             Desired outcome
------------------------------------------------------------------------------------------------
Expected Caller                 Allowed              -                            ✔
Expected Caller-Unknown API     Allowed              -                            ✘
Unwanted Caller #1              Blocked              API Gateway                  ✔
Unwanted Caller #2              Blocked              API Gateway                  ✔
Unwanted Caller #3              Allowed              -                            ✘
Unwanted Caller #4              Blocked              API Gateway                  ✔

> Finished scanning.
```

> **Improvement:** Unwanted Caller #4 is now blocked because its traffic does not originate from the designated ServiceA VPC Endpoint. Unwanted Caller #3 remains unblocked.

---

### Task 5 — Apply a VPC Endpoint Policy

Apply a restrictive policy to the `ServiceA-APIGW-EP` VPC endpoint to enforce identity-centric rules at the network boundary. This blocks unintended principals from making API calls through the endpoint, and restricts calls to only the intended resource.

1. Navigate to **VPC** → **Endpoints** → select **ServiceA-APIGW-EP**.
2. Choose the **Policy** tab → **Edit Policy** → select **Custom** → delete the existing policy.
3. Add the following policy, substituting the placeholder values:

```json
{
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "INSERT_SERVICEA_INSTANCE_ROLE_ARN"
            },
            "Action": "execute-api:Invoke",
            "Resource": "INSERT_API_METHOD_ARN"
        }
    ]
}
```

| Placeholder | Description |
|---|---|
| `INSERT_SERVICEA_INSTANCE_ROLE_ARN` | The IAM Role ARN attached to the `ServiceA-Instance` EC2 |
| `INSERT_API_METHOD_ARN` | The ARN of the specific API method (e.g. `arn:aws:execute-api:<region>:<account>:<api-id>/api/GET/orders`) |

4. Choose **Save**.

> This policy blocks API calls made by any principal other than the ServiceA instance IAM Role, and restricts access to only the ServiceB `GET /orders` endpoint.

#### Assessment Output (After VPC Endpoint Policy)

```
> Started scanning ...

Check                           Result               Enforced@             Desired outcome
------------------------------------------------------------------------------------------------
Expected Caller                 Allowed              -                            ✔
Expected Caller-Unknown API     Blocked              VPC endpoint                 ✔
Unwanted Caller #1              Blocked              VPC endpoint                 ✔
Unwanted Caller #2              Blocked              VPC endpoint                 ✔
Unwanted Caller #3              Blocked              VPC endpoint                 ✔
Unwanted Caller #4              Blocked              API Gateway                  ✔

> Finished scanning.
```

> **Improvement:** All 6 call patterns now meet the desired security outcome. The Expected Caller to the Unknown API is now correctly blocked at the VPC endpoint level.

---

### Task 6 — Tune the VPC Endpoint Security Group

Tighten the VPC endpoint security group to implement **defense in depth**. By narrowing the allowed source IP range, unwanted callers are blocked closer to the source, reducing unnecessary network traversal and downstream risk.

**Existing Rule:**
- HTTPS (TCP:443) from the entire ServiceA VPC CIDR `10.199.0.0/16` — overly broad.

**New Rule:**
- HTTPS (TCP:443) from the `ServiceASecurityGroup` security group only — scoped to the expected caller's security group.

1. Navigate to **VPC** → **Endpoints** → select **ServiceA-APIGW-EP** → **Security Groups** tab.
2. Choose the security group link → **Inbound rules** → **Edit inbound rules**.
3. Delete the existing rule, then choose **Add rule**:
   - **Type:** HTTPS
   - **Source:** Custom → select `ServiceASecurityGroup`
   - **Description:** `From ServiceA Instances Security Group`
4. Choose **Save rules**.

#### Assessment Output (After Security Group Tuning)

```
> Started scanning ...

Check                           Result               Enforced@             Desired outcome
------------------------------------------------------------------------------------------------
Expected Caller                 Allowed              -                            ✔
Expected Caller-Unknown API     Blocked              VPC endpoint                 ✔
Unwanted Caller #1              Blocked              Security Group               ✔
Unwanted Caller #2              Blocked              Security Group               ✔
Unwanted Caller #3              Blocked              Security Group               ✔
Unwanted Caller #4              Blocked              API Gateway                  ✔

> Finished scanning.
```

> **Improvement:** All security requirements are still met. Unwanted Callers #1, #2, and #3 are now blocked at the Security Group level — closer to the source — before they even reach API Gateway, reducing latency impact and cost of downstream enforcement.

---

## Security Posture Progression Summary

| Caller | Baseline | + IAM AuthZ | + Resource Policy | + VPC EP Policy | + SG Tuning |
|---|---|---|---|---|---|
| Expected Caller → ServiceB | ✅ Allow | ✅ Allow | ✅ Allow | ✅ Allow | ✅ Allow |
| Expected Caller → Unknown API | ❌ Allow | ❌ Allow | ❌ Allow | ✅ Block (VPC EP) | ✅ Block (VPC EP) |
| Unwanted Caller #1 | ✅ Block (APIGW) | ✅ Block (APIGW) | ✅ Block (APIGW) | ✅ Block (VPC EP) | ✅ Block (SG) |
| Unwanted Caller #2 | ❌ Allow | ✅ Block (APIGW) | ✅ Block (APIGW) | ✅ Block (VPC EP) | ✅ Block (SG) |
| Unwanted Caller #3 | ❌ Allow | ❌ Allow | ❌ Allow | ✅ Block (VPC EP) | ✅ Block (SG) |
| Unwanted Caller #4 | ❌ Allow | ❌ Allow | ✅ Block (APIGW) | ✅ Block (APIGW) | ✅ Block (APIGW) |

---

## Conclusion

By progressively applying layered security controls, this project demonstrates a complete Zero Trust implementation for service-to-service communication on AWS. The following was accomplished:

- ✅ Reviewed the current service-to-service architecture and identified its security gaps.
- ✅ Assessed the baseline security posture and found that only 2 of 6 call patterns were compliant.
- ✅ Enabled **IAM Authorization (SigV4)** on API Gateway to block unsigned requests.
- ✅ Updated the **API Gateway resource policy** to enforce trusted account and trusted VPC endpoint conditions.
- ✅ Applied a **VPC Endpoint policy** to restrict which IAM principals can call which API resources.
- ✅ Tuned the **VPC Endpoint security group** to enforce network-level controls closer to the source.

The final result is a fully compliant Zero Trust architecture where all 6 call patterns behave as intended.

---

## Additional Resources

- **Zero Trust architectures: An AWS perspective** — https://aws.amazon.com/security/zero-trust/
- **Private REST APIs in API Gateway** — https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-apis.html
- **AWS condition keys that can be used in API Gateway resource policies** — https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-resource-policies-aws-condition-keys.html
- **Authenticating Requests (AWS Signature Version 4)** — https://docs.aws.amazon.com/general/latest/gr/sigv4_signing.html
- **How API Gateway resource policies affect authorization workflow** — https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-resource-policies-iam-policies-interaction.html
- **API Gateway resource policy examples** — https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-resource-policies-examples.html
