---
layout: enterprise2
page_title: "Runs - API Docs - Terraform Enterprise"
sidebar_current: "docs-enterprise2-api-run"
---

# Runs API

-> **Note**: These API endpoints are in beta and are subject to change.

Performing a run on a new configuration is a multi step process.

1. [Create a configuration version on the workspace](./configuration-versions.html#create-a-configuration-version).
2. [Upload configuration files to the configuration version](./configuration-versions.html#upload-configuration-files).
3. [Create a Run on the workspace](#create-a-run); this is done automatically when a configuration file is uploaded.
4. [Create and queue an apply on the run](#apply); if auto-apply is not enabled.

Alternately, you can create a run with a pre-existing configuration version, even one from another workspace. This is useful for promoting known good code from one workspace to another.

## Create a Run

`POST /runs`

A run performs a plan and apply, using a configuration version and the workspace’s current variables. You can specify a configuration version when creating a run; if you don’t provide one, the run defaults to the workspace’s most recently used version. (A configuration version is “used” when it is created or used for a run in this workspace.)

### Request Body

This POST endpoint requires a JSON object with the following properties as a request payload.

Properties without a default value are required.

Key path                    | Type   | Default | Description
----------------------------|--------|---------|------------
`data.attributes.is-destroy` | bool | false | Specifies if this plan is a destroy plan, which will destroy all provisioned resources.
`data.attributes.message` | string | "Queued manually via the Terraform Enterprise API | Specifies the message to be associated with this run.
`data.relationships.workspace.data.id` | string | | Specifies the workspace ID where the run will be executed.
`data.relationships.configuration-version.data.id` | string | (nothing) | Specifies the configuration version to use for this run. If the `configuration-version` object is omitted, the run will be created using the workspace's latest configuration version.

Status  | Response                               | Reason
--------|----------------------------------------|-------
[200][] | [JSON API document][] (`type: "runs"`) | Successfully created a run
[404][] | [JSON API error object][]              | Organization or workspace not found, or user unauthorized to perform action
[422][] | [JSON API error object][]              | Validation errors

[JSON API document]: https://www.terraform.io/docs/enterprise/api/index.html#json-api-documents
[200]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200
[404]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404
[422]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/422


### Sample Payload

```json
{
  "data": {
    "attributes": {
      "is-destroy":false,
      "message": "Custom message"
    },
    "type":"runs",
    "relationships": {
      "workspace": {
        "data": {
          "type": "workspaces",
          "id": "ws-LLGHCr4SWy28wyGN"
        }
      },
      "configuration-version": {
        "data": {
          "type": "configuration-versions",
          "id": "cv-n4XQPBa2QnecZJ4G"
        }
      }
    }
  }
}
```

### Sample Request

```shell
curl \
  --header "Authorization: Bearer $ATLAS_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @payload.json \
  https://app.terraform.io/api/v2/runs
```

### Sample Response

```json
{
  "data": {
    "id": "run-CZcmD7eagjhyX0vN",
    "type": "runs",
    "attributes": {
      "auto-apply": false,
      "error-text": null,
      "is-destroy": false,
      "message": "Custom Message",
      "metadata": {},
      "source": "tfe-ui",
      "status": "pending",
      "status-timestamps": {},
      "terraform-version": "0.10.8",
      "created-at": "2017-11-29T19:56:15.205Z",
      "has-changes": false,
      "actions": {
        "is-cancelable": true,
        "is-confirmable": false,
        "is-discardable": false,
      },
      "permissions": {
        "can-apply": true,
        "can-cancel": true,
        "can-discard": true,
        "can-force-execute": true
      }
    },
    "relationships": {
      "apply": {...},
      "canceled-by": { ... },
      "configuration-version": {...},
      "confirmed-by": {...},
      "created-by": {...},
      "input-state-version": {...},
      "plan": {...},
      "run-events": {...},
      "policy-checks": {...},
      "workspace": {...},
      "comments": {...},
      "workspace-run-alerts": {...}
      }
    },
    "links": {
      "self": "/api/v2/runs/run-CZcmD7eagjhyX0vN"
    }
  }
}
```

## Apply a Run

`POST /runs/:run_id/actions/apply`

Parameter | Description
----------|------------
`run_id`  | The run ID to apply

Applies a run that is paused waiting for confirmation after a plan. This includes runs in the "needs confirmation" and "policy checked" states. This action is only required for workspaces without auto-apply enabled.

This endpoint queues the request to perform an apply; the apply might not happen immediately.

This endpoint represents an action as opposed to a resource. As such, the endpoint does not return any object in the response body.

Status  | Response                  | Reason(s)
--------|---------------------------|----------
[202][] | none                      | Successfully queued a discard request.
[409][] | [JSON API error object][] | Run was not paused for confirmation or priority; discard not allowed.

[202]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202
[409]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/409
[JSON API error object]: http://jsonapi.org/format/#error-objects


### Request Body

This POST endpoint allows an optional JSON object with the following properties as a request payload.

Key path  | Type   | Default | Description
----------|--------|---------|------------
`comment` | string | `null`  | An optional comment about the run.

### Sample Payload

This payload is optional, so the `curl` command will work without the `--data @payload.json` option too.

```json
{
  "comment":"Looks good to me"
}
```

### Sample Request

```shell
curl \
  --header "Authorization: Bearer $ATLAS_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @payload.json \
  https://app.terraform.io/api/v2/runs/run-DQGdmrWMX8z9yWQB/actions/apply
```


## List Runs in a Workspace

`GET /workspaces/:workspace_id/runs`

Parameter      | Description
---------------|------------
`workspace_id` | The workspace ID to list runs for.

### Sample Request

```shell
curl \
  --header "Authorization: Bearer $ATLAS_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  https://app.terraform.io/api/v2/workspaces/ws-yF7z4gyEQRhaCNG9/runs
```

### Sample Response

```json
{
  "data": [
    {
      "id": "run-bWSq4YeYpfrW4mx7",
      "type": "runs",
      "attributes": {
        "auto-apply": false,
        "error-text": null,
        "is-destroy": false,
        "message": "",
        "metadata": {},
        "source": "tfe-configuration-version",
        "status": "planned",
        "status-timestamps": {
          "planned-at": "2017-11-28T22:52:51+00:00"
        },
        "terraform-version": "0.11.0",
        "created-at": "2017-11-28T22:52:46.711Z",
        "has-changes": true,
        "actions": {
          "is-cancelable": false,
          "is-confirmable": true,
          "is-discardable": true,
        },
        "permissions": {
          "can-apply": true,
          "can-cancel": true,
          "can-discard": true,
          "can-force-execute": true
        }
      },
      "relationships": {
        "workspace": {...},
        "apply": {...},
        "canceled-by": {...},
        "configuration-version": {...},
        "confirmed-by": {...},
        "created-by": {...},
        "input-state-version": {...},
        "plan": {...},
        "run-events": {...},
        "policy-checks": {...},
        "comments": {...},
        "workspace-run-alerts": {...},
      "links": {
        "self": "/api/v2/runs/run-bWSq4YeYpfrW4mx7"
      }
    },
    ...
  ]
}
```

## Discard a Run

`POST /runs/:run_id/actions/discard`

Parameter | Description
----------|------------
`run_id`  | The run ID to discard

The `discard` action can be used to skip any remaining work on runs that are paused waiting for confirmation or priority. This includes runs in the "pending," "needs confirmation," "policy checked," and "policy override" states.

This endpoint queues the request to perform a discard; the discard might not happen immediately. After discarding, the run is completed and later runs can proceed.

This endpoint represents an action as opposed to a resource. As such, it does not return any object in the response body.

Status  | Response                  | Reason(s)
--------|---------------------------|----------
[202][] | none                      | Successfully queued a discard request.
[409][] | [JSON API error object][] | Run was not paused for confirmation or priority; discard not allowed.

### Request Body

This POST endpoint allows an optional JSON object with the following properties as a request payload.

Key path  | Type   | Default | Description
----------|--------|---------|------------
`comment` | string | `null`  | An optional explanation for why the run was discarded.


### Sample Payload

This payload is optional, so the `curl` command will work without the `--data @payload.json` option too.

```json
{
  "comment": "This run was discarded"
}
```

### Sample Request

```shell
curl \
  --header "Authorization: Bearer $ATLAS_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @payload.json \
  https://app.terraform.io/api/v2/runs/run-DQGdmrWMX8z9yWQB/actions/discard
```

## Cancel a Run

`POST /runs/:run_id/actions/cancel`

Parameter | Description
----------|------------
`run_id`  | The run ID to cancel

The `cancel` action can be used to interrupt a run that is currently planning or applying.

This endpoint queues the request to perform a cancel; the cancel might not happen immediately. After canceling, the run is completed and later runs can proceed.

This endpoint represents an action as opposed to a resource. As such, it does not return any object in the response body.

Status  | Response                  | Reason(s)
--------|---------------------------|----------
[202][] | none                      | Successfully queued a cancel request.
[409][] | [JSON API error object][] | Run was not planning or applying; cancel not allowed.

### Request Body

This POST endpoint allows an optional JSON object with the following properties as a request payload.

Key path  | Type   | Default | Description
----------|--------|---------|------------
`comment` | string | `null`  | An optional explanation for why the run was canceled.

### Sample Payload

This payload is optional, so the `curl` command will work without the `--data @payload.json` option too.

```json
{
  "comment": "This run was stuck and would never finish."
}
```

### Sample Request

```shell
curl \
  --header "Authorization: Bearer $ATLAS_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @payload.json \
  https://app.terraform.io/api/v2/runs/run-DQGdmrWMX8z9yWQB/actions/cancel
```

## Available Related Resources

The GET endpoints above can optionally return related resources, if requested with [the `include` query parameter](./index.html#inclusion-of-related-resources). The following resource types are available:

- `plan` - Additional information about plans.
- `apply` - Additional information about applies.
- `created_by` - Full user records of the users responsible for creating the runs.
- `configuration_version` - The configuration record used in the run.
- `configuration_version.ingress_attributes` - The commit information used in the run.
