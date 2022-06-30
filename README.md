# Overview
Looking into ability to show activity in terms of who is accessing dashboard and when. This is the end result of going through the steps below.

<img width="1602" alt="image" src="https://user-images.githubusercontent.com/100947826/176701552-c616be66-a897-43e0-b36d-1fefb91594e6.png">

# Environment
Elastic Cloud version 8.2.0

# Steps
## Enable Logging and Monitoring
Followed the [Enable logging and monitoring](https://www.elastic.co/guide/en/cloud/current/ec-enable-logging-and-monitoring.html) documentation to let Elasticsearch Service (ESS) take care of configuring the monitoring agents to ship to a deployment.

![image](https://user-images.githubusercontent.com/100947826/176703090-16f2c280-74ec-42e2-821e-3f2ba212f782.png)

## Enable Audit Logging
Even though logs are now flowing, you still have to turn on audit logging because it is disabled by default.  

This can be done by updating your [Kibana settings](https://www.elastic.co/guide/en/cloud/current/ec-manage-kibana-settings.html).  

![image](https://user-images.githubusercontent.com/100947826/176704636-4e699689-f886-449c-bffd-6ab8d82965a5.png)
![image](https://user-images.githubusercontent.com/100947826/176704696-3a490a1f-b67d-4c0f-be86-63ef8e73f429.png)

## Review Audit Data
One way to look at these logs is through Discover so create a Data View for "elastic-cloud-logs-*" as mentioned in the [Enable logging and monitoring](https://www.elastic.co/guide/en/cloud/current/ec-enable-logging-and-monitoring.html) documentation. Key fields to look at to filter the noise:
- kibana.saved_object.type: dashboard
- event.action: saved_object_resolve (this was important because before it was listing dashboards I hadn't actually opened)

![image](https://user-images.githubusercontent.com/100947826/176706066-a27488c1-c09a-40fd-8cc7-4795d86f8e8f.png)

Great, we can see who is accessing which dashboard and when. To complete the picture, would be nice to know which dashboard and which user instead of just their IDs.

## Enriching with Dashboard title
### Create a Dashboard Lookup Index
Found the dashboard titles tucked away in the hidden ".kibana" index.  As I didn't want to mess around with the index itself, I reindexed it and stashed the dashboard ID into the source of the destination index.

```
POST _reindex
{
  "source": {
    "index": ".kibana",
    "query": {
      "match": {
        "type": "dashboard"
      }
    }
  },
  "dest": {
    "index": "dashboard-lookup"
  },
  "script": {
    "source": "ctx._source.dashboard_id = ctx._id"
  }
}
```
### Setting Up an Ingest Pipeline
#### Getting the IDs to Match Up
There was a difference in the identifiers between the ".kibana" index and the audit logs ("cloud-elastic-log-*). 
- audit log shows --> "kibana.saved_object.id" : "77949db0-e1a3-11ec-80b0-21a99589f84a"
- kibana index shows --> "_id" : **dashboard:** 77949db0-e1a3-11ec-80b0-21a99589f84a

To deal with that, added a Script processor to prepend "dashboard" to the ID. Whenever new logs stream in, the ID needed for lookup goes into the **dashboard_id** field.
```
String prefix = 'dashboard:';
ctx['dashboard_id'] = prefix + ctx.kibana.saved_object.id
```
#### Add an Enrichment Policy
Building on that, this enrichment poicy does a lookup on the dashboard_id and then enriches the data with the dashboard title (e.g. NSF Proposals) that we're looking for.
```
PUT /_enrich/policy/dashboard-enrich
{
  "match": {
    "indices": "dashboard-lookup",
    "match_field": "dashboard_id",
    "enrich_fields": ["dashboard.title"]
  }
}
```
Adding that Enrich processor the the ingest pipeline looks like this:
![image](https://user-images.githubusercontent.com/100947826/176716274-5e95f97c-94fc-4cda-8772-f821435a4f8a.png)

#### Execute the Enrichment Policy
Don't forget, you have to execute the enrichment policy for it to take effect.
```
PUT /_enrich/policy/dashboard-enrich/_execute
```
### Updating the Index Template
We want this ingest pipeline to get invoked for any new documents indexed. Add it to the settings on the index template.

![image](https://user-images.githubusercontent.com/100947826/176717094-b0f4ba95-507e-477a-9e02-ff0b89c8da0e.png)

### Getting Data to Show
You may need to rollover the datastream to start getting the new fields and ingest pipeline humming.
```
POST elastic-cloud-logs-8/_rollover
```

Also you could do an **update_by_query** to update existing data.
```
POST /elastic-cloud-logs-8/_update_by_query?pipeline=dashboard-enrich
{
  "query": {
    "match": {
      "kibana.saved_object.type": "dashboard"
    }
  }
}
```
Still need to look into resolving user names but not too shabby...
![image](https://user-images.githubusercontent.com/100947826/176719770-08fc12df-0363-4680-bb30-d004351d5c63.png)

## Limitations
Because the dashboard lookups are reindexed from the .kibana index at a point in time, any new dashboards wouldn't show unless you reindex again.
