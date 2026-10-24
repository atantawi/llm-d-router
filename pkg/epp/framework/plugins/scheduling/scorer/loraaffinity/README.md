# LoRA Affinity Scorer Plugin

**Type:** `lora-affinity-scorer`

This plugin scores candidate endpoints based on whether a requested LoRA adapter is already loaded (or likely to load quickly) on each model server.

## What it does

For each candidate endpoint, the plugin checks endpoint metrics for the request's `targetModel` and assigns:

- `1.0`: target model is already active on endpoint (`ActiveModels` contains target)
- `0.8`: target model is not active, and fewer distinct adapters are active or waiting on the endpoint (the union of `ActiveModels` and `WaitingModels`) than `MaxActiveModels`
- `0.6`: target model is waiting (`WaitingModels` contains target), and the endpoint is not below that count
- `0.0`: none of the above

The checks run in that order. The `0.8` check counts adapters that have requests running or waiting, not adapters loaded on the GPU: vLLM reports an adapter only while it has requests, so an endpoint whose slots hold idle adapters still scores `0.8`.

Against current vLLM the `0.6` tier is unreachable. vLLM lists every adapter that has requests in both its running and its waiting label, so a waiting target is already in `ActiveModels` and scores `1.0`. The tier is reachable with the simulator and other backends.

## Scheduling intent

The scorer returns category `Affinity`, preferring endpoints with higher probability of immediate adapter reuse and lower model-load latency.

## Inputs consumed

The plugin consumes:

- `metrics.ActiveModelsKey` (`map[string]int`)
- `metrics.WaitingModelsKey` (`map[string]int`)

It also relies on the endpoint metric `MaxActiveModels` (vLLM's `max_lora` label) for the `0.8` check.

## Configuration

This scorer currently has no runtime parameters.

**Configuration Example:**
```yaml
plugins:
  - type: lora-affinity-scorer
    name: lora-affinity
schedulingProfiles:
  - name: default
    plugins:
      - pluginRef: lora-affinity
        weight: 1
```
