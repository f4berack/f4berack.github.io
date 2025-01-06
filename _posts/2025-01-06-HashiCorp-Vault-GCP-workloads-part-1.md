---
title: "(IN PROGRESS) HashiCorp Vault and GCP workloads (Part 1): a standardized process for secure credential retrieval"
published: true
---
&nbsp;

## Brief introduction

Hello world!\
This is my first blog article and I write to start discussing a solution, as per the title potentially standardizable (why not?), that I'm working on in order to retrieve securely any secret saved on self-hosted HashiCorp Vault instances from Google Cloud Platform (GCP) workloads, such as Cloud Run apps or Pyspark jobs running on Dataproc clusters.\
I would start by illustrating a primitive version in which an initial setup is done and by proposing a simple code template inherent to credential retrieval that is to be executed prior to the actual execution of a generic business logic requiring them.

## Initial Setup

First of all, check that HashiCorp Vault server is accessible from your chosen GCP execution environment and enable **IAM Service Account Credentials API** on your GCP project.\

Refer to the official documentation about **IAM Service Account Credentials API**:
[Google Cloud IAM Service Account Credentials API Documentation](https://cloud.google.com/iam/docs/reference/credentials/rest)

Next, you need to create a dedicated service account with **Service Account Token Creator** IAM role that impersonates itself.

Further details at the following page: [Create Short-Lived Credentials with Service Account Token Creator](https://cloud.google.com/iam/docs/create-short-lived-credentials-direct)

The final step involves enabling the GCP authentication method on your HashiCorp Vault server, please follow the official documentation for configuring this:
[Configure the GCP Authentication Method in HashiCorp Vault](https://developer.hashicorp.com/vault/docs/auth/gcp)


## Code template (python)


````python

import http.client
import ssl
import json
import datetime

from google.cloud import iam_credentials_v1

def get_credentials():

    now = datetime.datetime.now()
    exp = now + datetime.timedelta(minutes=<minutes>)

    # 1. Create JWT
    jwt = {
        "sub": "<your service account email address>",
        "aud": "vault/<your defined HashiCorp Vault role>",
        "exp": int(exp.timestamp()),
        "iat": int(now.timestamp()),
    }

    # 2. Sign JWT
    sign_jwt_request = iam_credentials_v1.SignJwtRequest(
        name="projects/-/serviceAccounts/<your service account email address>",
        payload=json.dumps(jwt),
    )

    sign_jwt_response = client.sign_jwt(request=sign_jwt_request)

    # 3. HashiCorp Vault GCP Auth
    vault_conn = http.client.HTTPSConnection(
        "<your HashiVorp Vault server URL>", context=ssl._create_unverified_context()
    )

    vault_auth_json_body = {"role": "<your defined HashiCorp Vault role>", "jwt": sign_jwt_response.signed_jwt}

    vault_conn.request("POST", "<base path>/auth/gcp/login", json.dumps(vault_auth_json_body))

    vault_response = vault_conn.getresponse()

    vault_response_data = json.loads(vault_response.read().decode("utf-8"))

    # 4. Get credentials
    headers = {"X-Vault-Token": vault_response_data["auth"]["client_token"]}

    vault_conn.request(
        "GET",
        "<base path>/<credential path>",
        headers=headers,
    )

    credentials_response = vault_conn.getresponse()

````

## Flowchart

![Vault GCP credentials retrieval flowchart image](/assets/Vault GCP credentials retrieval flowchart.png)

### Step 1
The JWT authentication token is created with the following claims:
- sub -> Use service account email address as subscriber
- aud -> Use the name of your defined role in HashiCorp Vault server
- iat
- exp

Further details at:
[Generating Jwts](https://developer.hashicorp.com/vault/docs/auth/gcp#generating-jwts)

### Step 2
Next, **signJwt** operation is done by calling its IAM Service Account Credentials endpoint, in order to sign the previously generated JWT.

Documentation:
[IAM Service Account Credentials signJwt](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts/signJwt)

"projects/-/serviceAccounts/<your service account email address>" is passed as **name** path param and a JSON Object containing all JWT claims as request payload.

### Step 3
Once you get the signed JWT, GCP authentication could be done by calling Vault GCP login API and in case of success a special token named **client_token** is returned.

Details: [Vault GCP Login](https://developer.hashicorp.com/vault/api-docs/auth/gcp#login)

### Step 4
As last step, credentials could be successfully retrieved and used by passing the client token in the custom HTTP header **X-Vault-Token**.

## Improvements

Just few ideas:
- Using hvac community library: [hvac](https://hvac.readthedocs.io/en/stable/overview.html)
- Preventing service account impersonation (workload identity federation?)

