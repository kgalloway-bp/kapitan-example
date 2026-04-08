# Kapitan Example


A minimal but realistic example of using [Kapitan](https://kapitan.dev) with a
**reclass** inventory hierarchy and **Jinja2** templates to generate Kubernetes
manifests from layered YAML inputs.

---

## Concepts

### The three inventory layers

Kapitan uses [reclass](https://reclass.pantsfree.com) under the hood to merge
inventory YAML files.  Each layer lives under `inventory/classes/` and is
composed left-to-right (rightmost wins on scalar conflicts; lists are merged):

```
Layer 1 – base/common.yml        ← lowest precedence, global defaults
Layer 2 – env/{dev,prod}.yml     ← environment overrides (replicas, resources, namespace)
Layer 3 – component/*.yml        ← component-specific values (image, port, env vars)
```

A **target** (under `inventory/targets/`) declares which classes to inherit,
and therefore which layers to merge:

```yaml
# inventory/targets/prod-frontend.yml
classes:
  - env.prod          # pulls in base.common transitively, then prod overrides
  - component.frontend
```

### Merge behaviour to pay attention to

| What changes            | Where it lives   | Effect                                               |
|-------------------------|------------------|------------------------------------------------------|
| `replicas`              | env layer        | dev=1, prod=3                                        |
| `image_pull_policy`     | env layer        | dev=IfNotPresent, prod=Always                        |
| `resources.*`           | env layer        | dev is lean; prod is production-sized                |
| `service.type`          | env layer        | dev=ClusterIP, prod=LoadBalancer                     |
| `service.type` override | component layer  | `worker` resets to ClusterIP even in prod (headless) |
| `deployment.env`        | component layer  | lists are appended; base provides `[]` as the anchor |

---

## Prerequisites

```bash
pip install kapitan
```

Kapitan requires Python 3.8+.  Tested with `kapitan==0.34.x`.

---

## Running

```bash
# Compile all targets
kapitan compile

# Compile a single target
kapitan compile -t dev-frontend

# Inspect the merged inventory for a target without compiling
kapitan inventory -t prod-worker
```

Compiled output lands in `compiled/<target-name>/manifests.yml` — a
two-document YAML stream (Deployment + Service) that can be applied directly:

```bash
kubectl apply -f compiled/prod-frontend/manifests.yml
```

---

## What each target produces

| Target          | namespace | replicas | image_pull_policy | service.type  | env vars |
|-----------------|-----------|----------|--------------------|---------------|----------|
| `dev-frontend`  | dev       | 1        | IfNotPresent       | ClusterIP     | —        |
| `dev-backend`   | dev       | 1        | IfNotPresent       | ClusterIP     | LOG_LEVEL, APP_PORT |
| `prod-frontend` | prod      | 3        | Always             | LoadBalancer  | —        |
| `prod-worker`   | prod      | 3        | Always             | ClusterIP (headless: clusterIP: None) | LOG_LEVEL, WORKER_CONCURRENCY, QUEUE_URL |

---

## Key Jinja2 template notes

The template receives the fully-merged `inventory` object from kapitan.
Access parameters via `inventory.parameters`:

```jinja2
{%- set p = inventory.parameters -%}
{{ p.deployment.replicas }}   {# → 3 for any prod target #}
{{ p.namespace }}              {# → "prod" or "dev" #}
```

Conditional blocks (e.g. env vars, headless clusterIP) use plain
`{% if %}...{% endif %}` without strip markers on the block tags so that
surrounding newlines are preserved in the YAML output:

```jinja2
{% if p.deployment.env | length > 0 %}
          env:
{% for env_var in p.deployment.env %}
            - name: {{ env_var.name }}
              value: "{{ env_var.value }}"
{% endfor %}
{% endif %}
```

> **Gotcha:** Kapitan's Jinja2 environment does not include Ansible's `quote`
> filter.  Use `"{{ value }}"` (YAML double-quotes around the interpolation)
> instead to force string typing on values like `"50m"` or `"8080"`.
