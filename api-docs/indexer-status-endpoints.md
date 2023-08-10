---
description: API endpoints for getting information about the ethscriptions.com indexer
---

# Indexer Status Endpoints



{% swagger method="get" path="/" baseUrl="/block_status" summary="Fetch Block Status of Indexer" expanded="true" %}
{% swagger-description %}
If your users are using the ethscriptions.com API to make important decisions it's recommended that you prevent them from doing so if the indexer is more than a few blocks behind.
{% endswagger-description %}

{% swagger-response status="200: OK" description="" %}
```json
{
  "current_block_number": 17749502,
  "last_imported_block": 17749501,
  "blocks_behind": 1
}
```
{% endswagger-response %}
{% endswagger %}

