## Step 0. Create a new namespace

```bash
NAMESPACE=llama-stack-fms
oc new-project $NAMESPACE
```
## Step 1. Deploy generation llm with vllm, for example

```bash
oc apply -k generation
```

## Step 2. Deploy hap detector with and configure it with the guardrails orchestrator

```bash
oc apply -k guardrails
```

## Step 3. Deploy lls

```bash
export FMS_ORCHESTRATOR_URL="https://$(oc get routes guardrails-orchestrator -o jsonpath='{.spec.host}')"
export VLLM_URL="http://$(oc get routes llm-route -o jsonpath='{.spec.host}')/v1"
envsubst < llsCR.yaml | oc apply -f -
```

## Optional: register a shield inside a pod

```bash
curl -X 'POST' \
  'http://localhost:8321/v1/shields' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "shield_id": "composite_shield",
    "provider_shield_id": "composite_shield",
    "provider_id": "trustyai_fms",
    "params": {
      "type": "content",
      "confidence_threshold": 0.5,
      "message_types": ["system"],
      "detectors": {
        "hap": {
          "detector_params": {}
        }
      }
    }
  }'
```

## Get the external route of the lls

```bash
oc expose service llamastack-custom-distribution-service --name=lls-route
LLS_ROUTE=$(oc get route lls-route -o jsonpath='{.spec.host}')
```

## Register a shield using the external route

```bash
curl -X 'POST' \
"http://$LLS_ROUTE/v1/shields" \
-H 'accept: application/json' \
-H 'Content-Type: application/json' \
-d '{
  "shield_id": "hap_shield",
  "provider_shield_id": "hap_shield",
  "provider_id": "trustyai_fms",
  "params": {
    "type": "content",
    "confidence_threshold": 0.5,
    "message_types": ["system"],
    "detectors": {
      "hap": {
        "detector_params": {}
      }
    }
  }
}'
```

## View registered shields using the external route

```bash
curl -s "http://$LLS_ROUTE/v1/shields" | jq '.'
```

## Send a message to the shield

```bash
curl -X POST "http://$LLS_ROUTE/v1/safety/run-shield" \
-H "Content-Type: application/json" \
-d '{
  "shield_id": "hap_shield",
  "messages": [
    {
      "content": "You dotard, I really hate this",
      "role": "system"
    }
  ]
}' | jq '.'
```