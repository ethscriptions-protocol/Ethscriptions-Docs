---
description: API endpoints for getting information about ethscriptions
---

# Ethscriptions Endpoints



{% swagger method="get" path="ethscriptions" baseUrl="/" summary="Get All Ethscriptions" expanded="true" %}
{% swagger-description %}
Returns list of all Ethscriptions within range of parameters
{% endswagger-description %}

{% swagger-parameter in="query" name="page" type="integer" %}

{% endswagger-parameter %}

{% swagger-parameter in="query" name="per_page" type="integer" %}

{% endswagger-parameter %}

{% swagger-parameter in="query" name="sort_order" type="string" %}
"asc" or "desc"
{% endswagger-parameter %}
{% endswagger %}



{% swagger method="get" path="ethscriptions/owned_by/:address" baseUrl="/" summary="Get All Ethscriptions by Address" expanded="true" %}
{% swagger-description %}
Returns list of Ethscriptions owned by specific ETH address
{% endswagger-description %}

{% swagger-parameter in="query" name="page" type="integer" %}

{% endswagger-parameter %}

{% swagger-parameter in="query" name="per_page" type="integer" %}

{% endswagger-parameter %}

{% swagger-parameter in="query" name="sort_order" type="string" %}
"asc" or "desc"
{% endswagger-parameter %}

{% swagger-parameter in="path" name="address" type="string" required="true" %}

{% endswagger-parameter %}
{% endswagger %}

{% swagger method="get" path="/:ethscription_id" baseUrl="/ethscriptions" summary="Get Specific Ethscription" expanded="true" %}
{% swagger-description %}
Returns JSON data of Ethscription
{% endswagger-description %}

{% swagger-parameter in="path" name="ethscription_id" required="true" %}
Transaction hash or Ethscription number
{% endswagger-parameter %}
{% endswagger %}



{% swagger method="get" path="/:ethscription_id/data" baseUrl="/ethscriptions" summary="Get data of an Ethscription" expanded="true" %}
{% swagger-description %}
Returns Ethscription's raw decoded data in the correct mimetype
{% endswagger-description %}

{% swagger-parameter in="path" name="ethscription_id" required="true" %}
Transaction hash of Ethscription number
{% endswagger-parameter %}
{% endswagger %}



{% swagger method="get" path="/exists/:sha256" baseUrl="/ethscriptions" summary="Check If Content Exists as Ethscription" expanded="true" %}
{% swagger-description %}

{% endswagger-description %}

{% swagger-parameter in="path" name="sha256" %}
sha256 hash of the UTF-8 dataURI
{% endswagger-parameter %}
{% endswagger %}

