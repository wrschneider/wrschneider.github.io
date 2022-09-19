---
layout: post
title: Use Power BI REST APIs to automate replacing Databricks and Snowflake credentials
description: Power BI REST APIs allow you to automate datasource credential replacement on deployment
tags: powerbi
---

Power BI provides a [comprehensive set of REST APIs](https://docs.microsoft.com/en-us/rest/api/power-bi/) for automating various
processes with Power BI.

One scenario is: you have a bunch of `pbix` files that define datasources and datasets, and you need to replace credentials with
different credentials in production.

You can automate credential replacement with Power BI REST API calls.

I will illustrate the raw REST calls with `az` and `curl` commands but Python, Powershell, etc. may be used for automation as well.

First step is to authenticate to Azure Active Directory with the user or service principal that you will use for Power BI calls.

```shell
az login --allow-no-subscriptions
```

Now get the access token you will need for Power BI API calls, and set it to an environment variable:

```shell
export PBI_TOKEN=$(az account get-access-token  --resource  "https://analysis.windows.net/powerbi/api"
   | jq --raw-output .accessToken)
```

Once you have the access token for Power BI calls, find the datasets with datasources you want to update:

```shell
# personal workspace
curl -v -X GET -H "Authorization: Bearer $PBI_TOKEN" -H "Content-Type: application/json" https://api.powerbi.com/v1.0/myorg/datasets/

# shared workspace
curl -v -X GET -H "Authorization: Bearer $PBI_TOKEN" -H "Content-Type: application/json" https://api.powerbi.com/v1.0/myorg/groups/{workspace-id}/datasets/
```

This will return a JSON response like this:

```json
{
  "@odata.context":"http://wabi-us-east2-c-primary-redirect.analysis.windows.net/v1.0/myorg/$metadata#datasets","value":[
    {
      "id":"xxxxx-xxxx-xxxxxxx","name":"DATASET NAME", ....
    },
    {
      "id":"xxxxx-xxxx-xxxxxxx","name":"DATASET NAME", ....
    },
    ...
  ]
}
```

Copy the ID for the dataset you want and now find the datasources for it:

```shell
# personal workspace
curl -v -X GET -H "Authorization: Bearer $PBI_TOKEN" -H "Content-Type: application/json" https://api.powerbi.com/v1.0/myorg/datasets/{dataset-id}/datasources

# shared workspace
curl -v -X GET -H "Authorization: Bearer $PBI_TOKEN" -H "Content-Type: application/json" https://api.powerbi.com/v1.0/myorg/groups/{workspace-id}/datasets/{dataset-id}/datasources
```

This will return a response like:

```json
{
    "@odata.context":"http://wabi-us-east2-c-primary-redirect.analysis.windows.net/v1.0/myorg/$metadata#datasources","value":[
    {
      "datasourceType":"Extension","connectionDetails":{
        "path":"DB_CONNECT_STRING","kind":"Snowflake"
      },"datasourceId":"xxxxxx-xxxx-xxxxxx","gatewayId":"xxxxx-xxxx-xxxxxx"
    }
    ]
}
```

Note the `datasourceId` and `gatewayId`.  Now make a call like this to see the datasource configuration:

```shell
# same for both personal and shared
curl -v -X GET -H "Authorization: Bearer $PBI_TOKEN" -H "Content-Type: application/json" https://api.powerbi.com/v1.0/myorg/gateways/{gateway-id}/datasources/{datasource-id}
```

And note the response will look something like this:

```json
{
  "@odata.context":"http://wabi-us-east2-c-primary-redirect.analysis.windows.net/v1.0/myorg/$metadata#gatewayDatasources/$entity","id":"xxxxx","gatewayId":"xxxxx","datasourceType":"Extension","connectionDetails":"{\"extensionDataSourceKind\":\"Databricks\",\"extensionDataSourcePath\":\"{\\\"host\\\":\\\"adb-xxxxx.xx.azuredatabricks.net\\\",\\\"httpPath\\\":\\\"\\\\/sql\\\\/1.0\\\\/warehouses\\\\/xxxxxx\\\"}\"}",
    "credentialType":"OAuth2", "credentialDetails":{
    "privacyLevel":"Organizational","useEndUserOAuth2Credentials":false
   }
}
```

Pay particular attention to the `credentialType` and `credentialDetails`.

Now we can [replace those credentials](https://learn.microsoft.com/en-us/rest/api/power-bi/gateways/update-datasource):

```shell
curl -v -X PATCH -H "Authorization: Bearer $PBI_TOKEN" -H "Content-Type: application/json" \
   https://api.powerbi.com/v1.0/myorg/gateways/{gateway-id}/datasources/{datasource-id}
   -d '{
            "credentialDetails": {
            "credentialType": "Key",
            "credentials": "{\"credentialData\":[{\"name\":\"key\", \"value\":\"dapi....=\"}]}",
            "encryptedConnection": "Encrypted",
            "encryptionAlgorithm": "None",
            "privacyLevel": "None"
            }
        }'
```

For Databricks personal access tokens you would use the `"Key"` credential type as above.

For Snowflake username/password access, you would follow the example for `Basic` credentials.

You can also follow the OAuth2 example, with an AAD token.  Note that the access tokens will be _different_ access tokens than the 
one you used for Power BI, as the audience (`aud`) is different:

```shell
export DB_TOKEN=$(az account get-access-token  --resource  "2ff814a6-3304-4ab8-85cb-cd0e6f879c1d"
   | jq --raw-output .accessToken)
```

The magic resource identifier 2ff... is described [in the Azure Databricks documentation](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/aad/service-prin-aad-token).  This doc also describes how to obtain the same token via `curl` directly.

If you are using Snowflake instead, the process is similar but you would use a different audience for the access token:

```shell
export SF_TOKEN=$(az account get-access-token  --resource  "https://analysis.windows.net/powerbi/connector/Snowflake" \
   | jq --raw-output .accessToken)
```

The audience value was obtained from [this document explaining how to connect Power BI and Snowflake](https://docs.snowflake.com/en/user-guide/oauth-powerbi.html)

The one caveat is, OAuth2 access tokens are short-lived (about an hour), so Power BI can no longer make DB connections after the access token expires.  There seems to be no way at present for the REST API to replicate the behavior of the Power BI service's web UI -- when you sign in with you AAD credentials through the browser, the credentials in Power BI are long-lived.
 This seems to be a [limitation with the Power BI REST API](https://community.powerbi.com/t5/Developer/Updating-OAuth-data-source-credentials-via-API-bearer-token/td-p/2028001) with an associated 
 [feature request](https://ideas.powerbi.com/ideas/idea/?ideaid=9f9b52f1-4428-ec11-b76a-281878bdb01d).

