---
layout: post
title: Power BI embed tokens, with row-level security and cross-report drillthroughs 
description: How to embed Power BI reports with row-level security and cross-report drillthroughs
---

Power BI provides REST APIs to generate embed tokens, which you can use to authenticate end users to Power
BI even if they are not authenticated to your Active Directory tenant.  This is known as [embed for your
customers](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-sample-for-customers?tabs=net-core): your application uses a service principal to call Power BI to generate an embed token, and
the end user can then use that embed token to execute Power BI reports.

The embedding process gives your application the ability to control fine-grained permissions.  If the user
should not have access to a particular report or dataset, the application can enforce this by not 
generating the embed token in the first place.

#### Embed tokens and row-level security
The embed token can convey information for row-level security (RLS)
via _effective identity_.  The effective 
identity specifies what roles to apply to the Power BI dataset, which may have defined filters within Power
BI.  The Power BI REST API document shows an example of [passing effective identity with roles](https://learn.microsoft.com/en-us/rest/api/power-bi/embed-token/reports-generate-token-in-group#generate-a-report-embed-token-using-an-effective-identity-example), when obtaining an embed token for a specific report.  The payload to the REST API would look something like
this:

```json
{
  "accessLevel": "View",
  "identities": [
    {
      "username": "username",
      "customData": "value", // optional, see
      "roles": [
        "rolename"
      ],
      "datasets": [
        "xxxxx-xxxx-xxxxxx"
      ]
    }
  ]
}
```

When the user executes the report, the dataset will be filtered according to the role `rolename`.

This is sufficient for some use cases, where the roles by themselves provide enough information to filter.

But what if the dimension for filtering is dynamic, and too granular to create a role for each value?  In 
this case, there may be some kind of lookup tables that specify what a user can see.  There are then two
ways you can apply that context in a Power BI filter:

* With a filter like `[user_column] = USERNAME()`.  With embedding the `USERNAME()` DAX function will 
return the `username` value your application passes in the embed API request payload.

* With `customData`.  This is a relatively new feature in Power BI, which [went GA in December 2021](https://powerbi.microsoft.com/en-sg/blog/the-customdata-feature-is-now-generally-available-in-power-bi/) (it was only available in Analysis Services previously).  This allows your application to explicitly pass a value to apply directly in the [embed token API request](https://learn.microsoft.com/en-us/rest/api/power-bi/embed-token/reports-generate-token-in-group#generate-a-report-embed-token-using-an-effective-identity-with-custom-data-for-azure-analysis-services-example) and then your Power BI dataset filter can use a DAX expression like 
  `[filter_column] = CUSTOMDATA()` or `[filter_column] = int(CUSTOMDATA())` 

Note that `CUSTOMDATA()` is always a string and may need to be converted to a different type before you can
filter.

The advantages of `customData` are
* your user security lookup table can update instantaneously, without having to worry about refresh in an imported model (may be less of an issue with composite models)
* filtering and retrieval of data is simplified especially with Direct Query (no extra join)
* your application can generate user access dynamically, even if there is no persistent lookup

#### Embed tokens and cross-report drillthroughs

Now that you have RLS working, how do you handle cross-report drillthroughs?

The key is to remember that Power BI will not go back to your application to generate a new embed token
for a drill-through.  So when you generate an embed token, it must cover not only the reports and 
datasets you want to display but _also_ any other reports or datasets you want to drill into.

The above examples showed the REST API for embedding a single report. There is [another REST API](https://learn.microsoft.com/en-us/rest/api/power-bi/embed-token/generate-token) 
that will allow you to grant access to _multiple_ reports and datasets with a single token.

The payload would look more like this:

```json
{
    "reports": [
      {"id": "aaaa-aaa-aaaaa"},
      {"id": "bbbb-bbb-bbbbb"},
      {"id": "cccc-ccc-ccccc"}
    ],
    "datasets": [
      {"id": "xxxx-xxx-xxxxx"},
      {"id": "yyyy-yyy-yyyyy"},
      {"id": "zzzz-zzz-zzzzz"}
    ],
    "identities": [
      {
        "username": "user",
        "roles": ["rolename"],
        "customData": "value",
        "datasets": ["xxxx-xxx-xxxxx", "yyyy-yyy-yyyyy"]
      },
      {
        "username": "user",
        "roles": ["rolename2"],
        "customData": "value2",
        "datasets": ["zzzz-zzz-zzzzz"]
      },
      
    ]
}
```
This will then enable access to all of the reports and datasets listed, including 
cross-report drillthroughs.

Note that two datasets can share the same effective identity, while others may use a different one.

Also, note that the context menu item for drillthrough will only display reports that were granted
access by the embed token.  If the embed token does not include any drill-through reports, the 
drill option will not display for the user at all!
