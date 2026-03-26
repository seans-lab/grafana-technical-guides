# Adding Annotations to a Grafana Dashboard Programmatically

**Version:** 1.0
**Audience:** Platform Engineers, Developers, SRE Teams
**Scope:** All methods for programmatically adding annotations to Grafana dashboards

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Method 1 — Grafana HTTP API](#api)
3. [Method 2 — Grafana Python Client](#python)
4. [Method 3 — Grafana Go Client](#go)
5. [Method 4 — Grafana Terraform Provider](#terraform)
6. [Method 5 — Grafana Alloy / Agent](#alloy)
7. [Method 6 — Grafana Provisioning](#provisioning)
8. [Method 7 — CI/CD Pipeline Integration](#cicd)
9. [Method 8 — Curl / Shell Scripts](#curl)
10. [Annotation Types & Use Cases](#types)
11. [Best Practices](#bestpractices)
12. [FAQs](#faqs)

---

<a name="overview"></a>
## 1. 🤔 Overview

### What Are Grafana Annotations?

Annotations are **markers on a dashboard timeline** that provide context for events. They appear as vertical lines or regions on panels and are useful for correlating metric changes with real-world events.

```
Dashboard Timeline
│
│    Deploy v1.2.3          Incident INC-042      Deploy v1.2.4
│         │                      │    │                │
│    ─────▼─────────────────────▼────▼────────────────▼─────
│         ↑                      ↑    ↑                ↑
│    Point annotation        Region annotation    Point annotation
│    (single timestamp)      (start + end time)  (single timestamp)
```

### Annotation Types

```
Point Annotation
  ├── Single timestamp
  ├── Marks a specific moment in time
  └── Examples: deployments, config changes, releases

Region Annotation
  ├── Start and end timestamp
  ├── Marks a duration of time
  └── Examples: incidents, maintenance windows, outages
```

### API Authentication Options

```
Option 1: Service Account Token (Recommended)
  Header: Authorization: Bearer <service-account-token>

Option 2: API Key (Legacy)
  Header: Authorization: Bearer <api-key>

Option 3: Basic Auth
  Header: Authorization: Basic <base64(user:password)>
```

---

<a name="api"></a>
## 2. 🌐 Method 1 — Grafana HTTP API

The Grafana HTTP API is the **most universal** method and underpins all other approaches.

### 2.1 — API Endpoints Reference

```
POST   /api/annotations              Create annotation
GET    /api/annotations              Query annotations
PATCH  /api/annotations/:id          Update annotation
DELETE /api/annotations/:id          Delete annotation
DELETE /api/annotations/by-region/:id Delete region annotation
POST   /api/annotations/graphite      Create Graphite annotation
```

---

### 2.2 — Create a Point Annotation

```bash
# Create a point annotation (single timestamp)
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-service-account-token>" \
  https://your-grafana.com/api/annotations \
  -d '{
    "dashboardUID": "abc123def456",
    "panelId": 1,
    "time": 1704067200000,
    "timeEnd": 1704067200000,
    "tags": ["deployment", "payments-api", "v1.2.3"],
    "text": "Deployed payments-api v1.2.3 to production"
  }'
```

#### Response

```json
{
  "id": 42,
  "message": "Annotation added"
}
```

---

### 2.3 — Create a Region Annotation (Start + End)

```bash
# Create a region annotation (time range)
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-service-account-token>" \
  https://your-grafana.com/api/annotations \
  -d '{
    "dashboardUID": "abc123def456",
    "panelId": 1,
    "time": 1704067200000,
    "timeEnd": 1704070800000,
    "tags": ["incident", "SEV-1", "payments"],
    "text": "INC-0234: Payment processing degraded"
  }'
```

---

### 2.4 — Create a Global Annotation (All Dashboards)

```bash
# Omit dashboardUID and panelId
# Annotation appears on ALL dashboards that
# have "Grafana" annotation datasource enabled

curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-service-account-token>" \
  https://your-grafana.com/api/annotations \
  -d '{
    "time": 1704067200000,
    "tags": ["deployment", "global"],
    "text": "Platform-wide deployment completed"
  }'
```

---

### 2.5 — Query Existing Annotations

```bash
# Query annotations by tag and time range
curl -X GET \
  -H "Authorization: Bearer <your-service-account-token>" \
  "https://your-grafana.com/api/annotations?\
from=1704067200000\
&to=1704153600000\
&tags=deployment\
&tags=payments-api\
&limit=100"
```

#### Response

```json
[
  {
    "id": 42,
    "alertId": 0,
    "alertName": "",
    "dashboardId": 5,
    "dashboardUID": "abc123def456",
    "panelId": 1,
    "userId": 0,
    "newState": "",
    "prevState": "",
    "created": 1704067200000,
    "updated": 1704067200000,
    "time": 1704067200000,
    "timeEnd": 1704067200000,
    "text": "Deployed payments-api v1.2.3 to production",
    "tags": ["deployment", "payments-api", "v1.2.3"],
    "login": "admin",
    "email": "admin@company.com",
    "avatarUrl": ""
  }
]
```

---

### 2.6 — Update an Annotation

```bash
# Update an existing annotation
curl -X PATCH \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-service-account-token>" \
  https://your-grafana.com/api/annotations/42 \
  -d '{
    "text": "Deployed payments-api v1.2.3 — ROLLED BACK",
    "tags": ["deployment", "payments-api", "v1.2.3", "rollback"]
  }'
```

---

### 2.7 — Delete an Annotation

```bash
# Delete an annotation by ID
curl -X DELETE \
  -H "Authorization: Bearer <your-service-account-token>" \
  https://your-grafana.com/api/annotations/42
```

---

### 2.8 — Complete API Payload Reference

```json
{
  "dashboardUID": "abc123def456",
  "dashboardId":  5,
  "panelId":      1,
  "time":         1704067200000,
  "timeEnd":      1704067200000,
  "isRegion":     false,
  "tags": [
    "deployment",
    "payments-api",
    "v1.2.3",
    "production"
  ],
  "text": "Human readable annotation text"
}
```

#### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `dashboardUID` | string | No | Target specific dashboard. Omit for global |
| `dashboardId` | integer | No | Legacy dashboard ID (prefer UID) |
| `panelId` | integer | No | Target specific panel. Omit for all panels |
| `time` | epoch ms | Yes | Annotation start time in milliseconds |
| `timeEnd` | epoch ms | No | End time for region annotations |
| `tags` | string[] | No | Array of tag strings for filtering |
| `text` | string | Yes | Annotation description |

---

<a name="python"></a>
## 3. 🐍 Method 2 — Grafana Python Client

### 3.1 — Using grafana-client Library

```bash
# Install the grafana-client library
pip install grafana-client
```

```python
# grafana_annotations.py
from grafana_client import GrafanaApi
from datetime import datetime, timezone
import time

# ============================================================
# Initialise Grafana client
# ============================================================
grafana = GrafanaApi.from_url(
    url="https://your-grafana.com",
    credential="your-service-account-token"
)

# ============================================================
# Helper: Convert datetime to epoch milliseconds
# ============================================================
def to_epoch_ms(dt: datetime) -> int:
    return int(dt.timestamp() * 1000)

def now_ms() -> int:
    return int(time.time() * 1000)

# ============================================================
# Create a deployment annotation
# ============================================================
def annotate_deployment(
    dashboard_uid: str,
    service: str,
    version: str,
    environment: str = "production",
    panel_id: int = None
) -> dict:
    """
    Create a point annotation marking a deployment event.

    Args:
        dashboard_uid: Target dashboard UID
        service:       Name of the service deployed
        version:       Version deployed (e.g. v1.2.3)
        environment:   Target environment
        panel_id:      Optional specific panel ID

    Returns:
        API response dict with annotation ID
    """
    payload = {
        "dashboardUID": dashboard_uid,
        "time": now_ms(),
        "timeEnd": now_ms(),
        "tags": [
            "deployment",
            service,
            version,
            environment
        ],
        "text": f"Deployed {service} {version} to {environment}"
    }

    if panel_id:
        payload["panelId"] = panel_id

    return grafana.annotations.add_annotation(payload)


# ============================================================
# Create an incident region annotation
# ============================================================
def annotate_incident_start(
    dashboard_uid: str,
    incident_id: str,
    title: str,
    severity: str,
    services: list
) -> int:
    """
    Mark the start of an incident.
    Returns the annotation ID for later closure.
    """
    payload = {
        "dashboardUID": dashboard_uid,
        "time": now_ms(),
        "timeEnd": now_ms(),   # Will be updated on resolution
        "tags": [
            "incident",
            incident_id,
            severity,
            *services
        ],
        "text": f"[{severity.upper()}] {incident_id}: {title} — OPEN"
    }

    response = grafana.annotations.add_annotation(payload)
    return response["id"]


def annotate_incident_resolved(
    annotation_id: int,
    resolution_notes: str = ""
) -> dict:
    """
    Update an incident annotation with the resolution time.
    """
    update_payload = {
        "timeEnd": now_ms(),
        "text": f"RESOLVED — {resolution_notes}",
        "tags": ["incident", "resolved"]
    }

    return grafana.annotations.update_annotation(
        annotation_id,
        update_payload
    )


# ============================================================
# Create a maintenance window annotation
# ============================================================
def annotate_maintenance_window(
    dashboard_uid: str,
    start_time: datetime,
    end_time: datetime,
    description: str,
    services: list
) -> dict:
    """
    Create a region annotation for a maintenance window.
    """
    payload = {
        "dashboardUID": dashboard_uid,
        "time": to_epoch_ms(start_time),
        "timeEnd": to_epoch_ms(end_time),
        "tags": [
            "maintenance",
            *services
        ],
        "text": f"Maintenance Window: {description}"
    }

    return grafana.annotations.add_annotation(payload)


# ============================================================
# Query annotations
# ============================================================
def get_annotations(
    dashboard_uid: str,
    tags: list = None,
    from_time: datetime = None,
    to_time: datetime = None,
    limit: int = 100
) -> list:
    """
    Query annotations for a dashboard with optional filters.
    """
    params = {
        "dashboardUID": dashboard_uid,
        "limit": limit
    }

    if tags:
        params["tags"] = tags

    if from_time:
        params["from"] = to_epoch_ms(from_time)

    if to_time:
        params["to"] = to_epoch_ms(to_time)

    return grafana.annotations.get_annotations(**params)


# ============================================================
# Example usage
# ============================================================
if __name__ == "__main__":

    DASHBOARD_UID = "abc123def456"

    # Mark a deployment
    result = annotate_deployment(
        dashboard_uid=DASHBOARD_UID,
        service="payments-api",
        version="v1.2.3",
        environment="production"
    )
    print(f"Deployment annotation created: ID {result['id']}")

    # Mark incident start
    incident_annotation_id = annotate_incident_start(
        dashboard_uid=DASHBOARD_UID,
        incident_id="INC-0234",
        title="Payment processing degraded",
        severity="SEV-1",
        services=["payments-api", "payments-database"]
    )
    print(f"Incident annotation created: ID {incident_annotation_id}")

    # Later — mark incident resolved
    annotate_incident_resolved(
        annotation_id=incident_annotation_id,
        resolution_notes="Connection pool limit increased"
    )
    print("Incident resolved annotation updated")
```

---

### 3.2 — Using requests Library (No Dependencies)

```python
# grafana_annotations_requests.py
# Uses only the standard requests library

import requests
import time
from datetime import datetime
from typing import Optional, List

class GrafanaAnnotations:
    """
    Grafana Annotations client using requests library.
    No external dependencies beyond requests.
    """

    def __init__(
        self,
        grafana_url: str,
        token: str,
        verify_ssl: bool = True
    ):
        self.base_url = grafana_url.rstrip("/")
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
            "Accept": "application/json"
        })
        self.verify_ssl = verify_ssl

    def _request(
        self,
        method: str,
        endpoint: str,
        **kwargs
    ) -> dict:
        url = f"{self.base_url}{endpoint}"
        response = self.session.request(
            method,
            url,
            verify=self.verify_ssl,
            **kwargs
        )
        response.raise_for_status()
        return response.json()

    def create(
        self,
        text: str,
        tags: List[str],
        time_ms: Optional[int] = None,
        time_end_ms: Optional[int] = None,
        dashboard_uid: Optional[str] = None,
        panel_id: Optional[int] = None
    ) -> dict:
        """Create an annotation."""
        payload = {
            "text": text,
            "tags": tags,
            "time": time_ms or int(time.time() * 1000),
            "timeEnd": time_end_ms or int(time.time() * 1000)
        }

        if dashboard_uid:
            payload["dashboardUID"] = dashboard_uid

        if panel_id is not None:
            payload["panelId"] = panel_id

        return self._request("POST", "/api/annotations", json=payload)

    def update(self, annotation_id: int, **kwargs) -> dict:
        """Update an existing annotation."""
        return self._request(
            "PATCH",
            f"/api/annotations/{annotation_id}",
            json=kwargs
        )

    def delete(self, annotation_id: int) -> dict:
        """Delete an annotation."""
        return self._request(
            "DELETE",
            f"/api/annotations/{annotation_id}"
        )

    def query(
        self,
        dashboard_uid: Optional[str] = None,
        tags: Optional[List[str]] = None,
        from_ms: Optional[int] = None,
        to_ms: Optional[int] = None,
        limit: int = 100
    ) -> list:
        """Query annotations with filters."""
        params = {"limit": limit}

        if dashboard_uid:
            params["dashboardUID"] = dashboard_uid

        if tags:
            params["tags"] = tags

        if from_ms:
            params["from"] = from_ms

        if to_ms:
            params["to"] = to_ms

        return self._request(
            "GET",
            "/api/annotations",
            params=params
        )


# ============================================================
# Usage example
# ============================================================
if __name__ == "__main__":
    import os

    client = GrafanaAnnotations(
        grafana_url=os.environ["GRAFANA_URL"],
        token=os.environ["GRAFANA_TOKEN"]
    )

    # Create deployment annotation
    result = client.create(
        text="Deployed payments-api v1.2.3",
        tags=["deployment", "payments-api", "v1.2.3"],
        dashboard_uid="abc123def456"
    )

    print(f"Created annotation ID: {result['id']}")
```

---

<a name="go"></a>
## 4. 🐹 Method 3 — Grafana Go Client

```go
// grafana_annotations.go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "time"

    goapi "github.com/grafana/grafana-openapi-client-go/client"
    "github.com/grafana/grafana-openapi-client-go/client/annotations"
    "github.com/grafana/grafana-openapi-client-go/models"
)

// GrafanaAnnotations wraps the Grafana API client
type GrafanaAnnotations struct {
    client *goapi.GrafanaHTTPAPI
}

// NewGrafanaAnnotations creates a new annotations client
func NewGrafanaAnnotations(
    grafanaURL string,
    token string,
) *GrafanaAnnotations {

    cfg := goapi.DefaultTransportConfig().
        WithHost(grafanaURL).
        WithBasePath("/api").
        WithSchemes([]string{"https"})

    client := goapi.NewHTTPClientWithConfig(nil, cfg)

    // Set auth header
    client.SetTransport(
        &authTransport{
            token:   token,
            wrapped: client.Transport,
        },
    )

    return &GrafanaAnnotations{client: client}
}

// nowMs returns current time in milliseconds
func nowMs() int64 {
    return time.Now().UnixMilli()
}

// toEpochMs converts time.Time to epoch milliseconds
func toEpochMs(t time.Time) int64 {
    return t.UnixMilli()
}

// CreateDeploymentAnnotation marks a deployment event
func (g *GrafanaAnnotations) CreateDeploymentAnnotation(
    ctx context.Context,
    dashboardUID string,
    service string,
    version string,
    environment string,
) (int64, error) {

    now := nowMs()
    text := fmt.Sprintf(
        "Deployed %s %s to %s",
        service, version, environment,
    )

    tags := []string{
        "deployment",
        service,
        version,
        environment,
    }

    params := annotations.NewCreateAnnotationParams().
        WithContext(ctx).
        WithBody(&models.PostAnnotationsCmd{
            DashboardUID: dashboardUID,
            Time:         now,
            TimeEnd:      now,
            Tags:         tags,
            Text:         text,
        })

    resp, err := g.client.Annotations.CreateAnnotation(params)
    if err != nil {
        return 0, fmt.Errorf("creating annotation: %w", err)
    }

    return resp.Payload.ID, nil
}

// CreateIncidentAnnotation marks an incident start
func (g *GrafanaAnnotations) CreateIncidentAnnotation(
    ctx context.Context,
    dashboardUID string,
    incidentID string,
    title string,
    severity string,
    services []string,
) (int64, error) {

    now := nowMs()
    text := fmt.Sprintf(
        "[%s] %s: %s — OPEN",
        severity, incidentID, title,
    )

    tags := append([]string{"incident", incidentID, severity}, services...)

    params := annotations.NewCreateAnnotationParams().
        WithContext(ctx).
        WithBody(&models.PostAnnotationsCmd{
            DashboardUID: dashboardUID,
            Time:         now,
            TimeEnd:      now,
            Tags:         tags,
            Text:         text,
        })

    resp, err := g.client.Annotations.CreateAnnotation(params)
    if err != nil {
        return 0, fmt.Errorf("creating incident annotation: %w", err)
    }

    return resp.Payload.ID, nil
}

// ResolveIncidentAnnotation updates incident annotation with end time
func (g *GrafanaAnnotations) ResolveIncidentAnnotation(
    ctx context.Context,
    annotationID int64,
    resolutionNotes string,
) error {

    now := nowMs()
    text := fmt.Sprintf("RESOLVED — %s", resolutionNotes)

    params := annotations.NewPatchAnnotationParams().
        WithContext(ctx).
        WithAnnotationID(annotationID).
        WithBody(&models.PatchAnnotationsCmd{
            TimeEnd: now,
            Text:    text,
            Tags:    []string{"incident", "resolved"},
        })

    _, err := g.client.Annotations.PatchAnnotation(params)
    if err != nil {
        return fmt.Errorf("resolving annotation: %w", err)
    }

    return nil
}

// CreateMaintenanceWindow creates a region annotation
func (g *GrafanaAnnotations) CreateMaintenanceWindow(
    ctx context.Context,
    dashboardUID string,
    start time.Time,
    end time.Time,
    description string,
    services []string,
) (int64, error) {

    tags := append([]string{"maintenance"}, services...)
    text := fmt.Sprintf("Maintenance Window: %s", description)

    params := annotations.NewCreateAnnotationParams().
        WithContext(ctx).
        WithBody(&models.PostAnnotationsCmd{
            DashboardUID: dashboardUID,
            Time:         toEpochMs(start),
            TimeEnd:      toEpochMs(end),
            Tags:         tags,
            Text:         text,
        })

    resp, err := g.client.Annotations.CreateAnnotation(params)
    if err != nil {
        return 0, fmt.Errorf("creating maintenance annotation: %w", err)
    }

    return resp.Payload.ID, nil
}

func main() {
    ctx := context.Background()

    client := NewGrafanaAnnotations(
        os.Getenv("GRAFANA_URL"),
        os.Getenv("GRAFANA_TOKEN"),
    )

    // Create deployment annotation
    id, err := client.CreateDeploymentAnnotation(
        ctx,
        "abc123def456",
        "payments-api",
        "v1.2.3",
        "production",
    )
    if err != nil {
        log.Fatalf("Failed to create annotation: %v", err)
    }

    fmt.Printf("Created annotation ID: %d\n", id)
}
```

---

<a name="terraform"></a>
## 5. 🏗️ Method 4 — Grafana Terraform Provider

```hcl
# annotations.tf
# Manage annotations as Infrastructure as Code

terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = ">= 2.0.0"
    }
  }
}

provider "grafana" {
  url  = var.grafana_url
  auth = var.grafana_token
}

# ============================================================
# Variables
# ============================================================
variable "grafana_url" {
  description = "Grafana instance URL"
  type        = string
}

variable "grafana_token" {
  description = "Grafana service account token"
  type        = string
  sensitive   = true
}

variable "dashboard_uid" {
  description = "Target dashboard UID"
  type        = string
  default     = "abc123def456"
}

# ============================================================
# Get current time in milliseconds (Terraform workaround)
# ============================================================
resource "time_static" "deployment_time" {}

locals {
  now_ms = parseint(
    format("%d000",
      time_static.deployment_time.unix
    ), 10
  )
}

# ============================================================
# Deployment annotation
# ============================================================
resource "grafana_annotation" "deployment" {
  dashboard_uid = var.dashboard_uid
  panel_id      = 1

  time     = local.now_ms
  time_end = local.now_ms

  text = "Deployed payments-api v${var.version} to production"
  tags = [
    "deployment",
    "payments-api",
    "v${var.version}",
    "production",
    "terraform"
  ]
}

# ============================================================
# Maintenance window annotation (region)
# ============================================================
resource "grafana_annotation" "maintenance_window" {
  dashboard_uid = var.dashboard_uid

  # Start: now
  time = local.now_ms

  # End: now + 2 hours (7200000 ms)
  time_end = local.now_ms + 7200000

  text = "Scheduled maintenance window — database upgrade"
  tags = [
    "maintenance",
    "planned",
    "payments-database"
  ]
}

# ============================================================
# Global annotation (visible on all dashboards)
# ============================================================
resource "grafana_annotation" "global_deployment" {
  # No dashboard_uid = global annotation
  time     = local.now_ms
  time_end = local.now_ms

  text = "Platform-wide infrastructure upgrade"
  tags = [
    "infrastructure",
    "deployment",
    "global"
  ]
}

# ============================================================
# Outputs
# ============================================================
output "deployment_annotation_id" {
  value = grafana_annotation.deployment.id
}
```

---

<a name="alloy"></a>
## 6. 🔄 Method 5 — Grafana Alloy

Use Grafana Alloy to automatically create annotations when specific events occur in your telemetry pipeline:

```river
// annotations-pipeline.alloy
// Automatically create Grafana annotations from telemetry events

// ============================================================
// Watch for deployment events in logs
// and create annotations automatically
// ============================================================
loki.source.kubernetes "app_logs" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.process.watch_deployments.receiver]
}

loki.process "watch_deployments" {

  // Match deployment completion log lines
  stage.match {
    selector = "{app=\"deployment-manager\"}"

    stage.json {
      expressions = {
        event   = "event",
        service = "service",
        version = "version",
        env     = "environment"
      }
    }

    stage.match {
      selector = "{event=\"deployment_complete\"}"

      // Fire annotation via remote_write webhook
      stage.output {
        source = "message"
      }
    }
  }

  forward_to = [loki.write.grafana_cloud.receiver]
}

// ============================================================
// HTTP component to create annotations via Grafana API
// Triggered by external events
// ============================================================
prometheus.exporter.self "alloy_self" {}

// Use remote.http to POST annotations when events occur
remote.http "deployment_annotation" {
  url            = env("GRAFANA_URL") + "/api/annotations"
  method         = "POST"

  headers = {
    "Authorization" = "Bearer " + env("GRAFANA_TOKEN"),
    "Content-Type"  = "application/json",
  }

  body = json_encode({
    "dashboardUID" = env("DASHBOARD_UID"),
    "time"         = sys.env("ANNOTATION_TIME"),
    "timeEnd"      = sys.env("ANNOTATION_TIME"),
    "tags"         = ["deployment", "alloy-triggered"],
    "text"         = "Deployment detected by Alloy pipeline"
  })
}
```

---

<a name="cicd"></a>
## 7. 🚀 Method 7 — CI/CD Pipeline Integration

### 7.1 — GitHub Actions

```yaml
# .github/workflows/deploy-with-annotations.yml
name: Deploy with Grafana Annotations

on:
  push:
    branches: [main]

env:
  GRAFANA_URL: ${{ secrets.GRAFANA_URL }}
  GRAFANA_TOKEN: ${{ secrets.GRAFANA_TOKEN }}
  DASHBOARD_UID: ${{ vars.PAYMENTS_DASHBOARD_UID }}
  SERVICE_NAME: payments-api

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Get version
        id: version
        run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT

      # ========================================================
      # Create deployment START annotation
      # ========================================================
      - name: Annotate deployment start
        id: annotate_start
        run: |
          RESPONSE=$(curl -s -X POST \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
            "${GRAFANA_URL}/api/annotations" \
            -d "{
              \"dashboardUID\": \"${DASHBOARD_UID}\",
              \"time\": $(date +%s%3N),
              \"timeEnd\": $(date +%s%3N),
              \"tags\": [
                \"deployment\",
                \"${SERVICE_NAME}\",
                \"${{ steps.version.outputs.version }}\",
                \"in-progress\"
              ],
              \"text\": \"🚀 Deploying ${SERVICE_NAME} ${{ steps.version.outputs.version }}\"
            }")

          ANNOTATION_ID=$(echo $RESPONSE | jq -r '.id')
          echo "annotation_id=${ANNOTATION_ID}" >> $GITHUB_OUTPUT
          echo "Created annotation ID: ${ANNOTATION_ID}"

      # ========================================================
      # Your deployment step
      # ========================================================
      - name: Deploy application
        run: |
          echo "Deploying ${{ env.SERVICE_NAME }}..."
          # kubectl apply -f k8s/
          # or helm upgrade...

      # ========================================================
      # Update annotation on SUCCESS
      # ========================================================
      - name: Annotate deployment success
        if: success()
        run: |
          curl -s -X PATCH \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
            "${GRAFANA_URL}/api/annotations/${{ steps.annotate_start.outputs.annotation_id }}" \
            -d "{
              \"tags\": [
                \"deployment\",
                \"${SERVICE_NAME}\",
                \"${{ steps.version.outputs.version }}\",
                \"success\"
              ],
              \"text\": \"✅ Deployed ${SERVICE_NAME} ${{ steps.version.outputs.version }} — SUCCESS\"
            }"

      # ========================================================
      # Update annotation on FAILURE
      # ========================================================
      - name: Annotate deployment failure
        if: failure()
        run: |
          curl -s -X PATCH \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
            "${GRAFANA_URL}/api/annotations/${{ steps.annotate_start.outputs.annotation_id }}" \
            -d "{
              \"tags\": [
                \"deployment\",
                \"${SERVICE_NAME}\",
                \"${{ steps.version.outputs.version }}\",
                \"failure\"
              ],
              \"text\": \"❌ FAILED: Deploying ${SERVICE_NAME} ${{ steps.version.outputs.version }}\"
            }"
```

---

### 7.2 — Azure DevOps Pipeline

```yaml
# azure-pipelines-annotations.yml
trigger:
  branches:
    include: [main]

variables:
  SERVICE_NAME: payments-api
  GRAFANA_URL: $(GRAFANA_URL_SECRET)
  GRAFANA_TOKEN: $(GRAFANA_TOKEN_SECRET)
  DASHBOARD_UID: $(PAYMENTS_DASHBOARD_UID)

stages:
  - stage: Deploy
    jobs:
      - job: DeployWithAnnotations
        steps:

          - task: Bash@3
            name: AnnotateStart
            displayName: Create deployment start annotation
            inputs:
              targetType: inline
              script: |
                RESPONSE=$(curl -s -X POST \
                  -H "Content-Type: application/json" \
                  -H "Authorization: Bearer $(GRAFANA_TOKEN)" \
                  "$(GRAFANA_URL)/api/annotations" \
                  -d "{
                    \"dashboardUID\": \"$(DASHBOARD_UID)\",
                    \"time\": $(date +%s%3N),
                    \"timeEnd\": $(date +%s%3N),
                    \"tags\": [
                      \"deployment\",
                      \"$(SERVICE_NAME)\",
                      \"$(Build.BuildNumber)\"
                    ],
                    \"text\": \"Deploying $(SERVICE_NAME) build $(Build.BuildNumber)\"
                  }")

                ANNOTATION_ID=$(echo $RESPONSE | python3 -c \
                  "import sys,json; print(json.load(sys.stdin)['id'])")

                echo "##vso[task.setvariable variable=annotationId;isOutput=true]\
                  ${ANNOTATION_ID}"

          - task: Bash@3
            displayName: Deploy application
            inputs:
              targetType: inline
              script: |
                echo "Deploying application..."
                # Your deployment commands here

          - task: Bash@3
            displayName: Update annotation on success
            condition: succeeded()
            inputs:
              targetType: inline
              script: |
                curl -s -X PATCH \
                  -H "Content-Type: application/json" \
                  -H "Authorization: Bearer $(GRAFANA_TOKEN)" \
                  "$(GRAFANA_URL)/api/annotations/$(AnnotateStart.annotationId)" \
                  -d "{
                    \"tags\": [\"deployment\", \"$(SERVICE_NAME)\", \"success\"],
                    \"text\": \"✅ Deployed $(SERVICE_NAME) $(Build.BuildNumber)\"
                  }"
```

---

<a name="curl"></a>
## 8. 🖥️ Method 8 — Curl / Shell Scripts

### 8.1 — Reusable Shell Function Library

```bash
#!/bin/bash
# grafana-annotations.sh
# Source this file to use annotation functions
# source ./grafana-annotations.sh

# ============================================================
# Configuration — set these or pass as environment variables
# ============================================================
GRAFANA_URL="${GRAFANA_URL:-https://your-grafana.com}"
GRAFANA_TOKEN="${GRAFANA_TOKEN:-your-token-here}"

# ============================================================
# Helper functions
# ============================================================
now_ms() {
  echo $(($(date +%s) * 1000))
}

# ============================================================
# Create a point annotation
# ============================================================
grafana_annotate() {
  local text="$1"
  local tags_json="$2"        # JSON array string: '["tag1","tag2"]'
  local dashboard_uid="$3"    # Optional
  local panel_id="$4"         # Optional

  local time_ms
  time_ms=$(now_ms)

  local payload
  payload=$(cat <<EOF
{
  "time": ${time_ms},
  "timeEnd": ${time_ms},
  "tags": ${tags_json},
  "text": "${text}"
EOF
  )

  # Add optional fields
  if [ -n "$dashboard_uid" ]; then
    payload=$(echo "$payload" | \
      jq --arg uid "$dashboard_uid" \
      '. + {"dashboardUID": $uid}')
  fi

  if [ -n "$panel_id" ]; then
    payload=$(echo "$payload" | \
      jq --argjson pid "$panel_id" \
      '. + {"panelId": $pid}')
  fi

  # Close JSON
  payload="${payload}}"

  local response
  response=$(curl -s \
    -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
    "${GRAFANA_URL}/api/annotations" \
    -d "$payload")

  local annotation_id
  annotation_id=$(echo "$response" | jq -r '.id')

  if [ "$annotation_id" == "null" ] || [ -z "$annotation_id" ]; then
    echo "ERROR: Failed to create annotation" >&2
    echo "Response: $response" >&2
    return 1
  fi

  echo "$annotation_id"
}

# ============================================================
# Create a region annotation (start + end time)
# ============================================================
grafana_annotate_region() {
  local text="$1"
  local tags_json="$2"
  local start_ms="$3"
  local end_ms="$4"
  local dashboard_uid="$5"

  local payload
  payload=$(jq -n \
    --arg text "$text" \
    --argjson tags "$tags_json" \
    --argjson start "$start_ms" \
    --argjson end "$end_ms" \
    '{
      text: $text,
      tags: $tags,
      time: $start,
      timeEnd: $end
    }')

  if [ -n "$dashboard_uid" ]; then
    payload=$(echo "$payload" | \
      jq --arg uid "$dashboard_uid" \
      '. + {"dashboardUID": $uid}')
  fi

  curl -s \
    -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
    "${GRAFANA_URL}/api/annotations" \
    -d "$payload" | jq -r '.id'
}

# ============================================================
# Update an existing annotation
# ============================================================
grafana_annotate_update() {
  local annotation_id="$1"
  local text="$2"
  local tags_json="$3"

  local payload
  payload=$(jq -n \
    --arg text "$text" \
    --argjson tags "$tags_json" \
    '{text: $text, tags: $tags}')

  curl -s \
    -X PATCH \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
    "${GRAFANA_URL}/api/annotations/${annotation_id}" \
    -d "$payload"
}

# ============================================================
# Deployment annotation helper
# ============================================================
annotate_deployment() {
  local service="$1"
  local version="$2"
  local environment="${3:-production}"
  local dashboard_uid="$4"

  local tags_json
  tags_json=$(jq -n \
    --arg s "$service" \
    --arg v "$version" \
    --arg e "$environment" \
    '["deployment", $s, $v, $e]')

  local id
  id=$(grafana_annotate \
    "Deployed ${service} ${version} to ${environment}" \
    "$tags_json" \
    "$dashboard_uid")

  echo "Deployment annotation created: ID ${id}"
  echo "$id"
}

# ============================================================
# Incident annotation helper
# ============================================================
annotate_incident_open() {
  local incident_id="$1"
  local title="$2"
  local severity="$3"
  local dashboard_uid="$4"

  local tags_json
  tags_json=$(jq -n \
    --arg id "$incident_id" \
    --arg sev "$severity" \
    '["incident", $id, $sev, "open"]')

  local annotation_id
  annotation_id=$(grafana_annotate \
    "[${severity}] ${incident_id}: ${title}" \
    "$tags_json" \
    "$dashboard_uid")

  echo "$annotation_id"
}

annotate_incident_close() {
  local annotation_id="$1"
  local resolution="$2"

  grafana_annotate_update \
    "$annotation_id" \
    "RESOLVED: ${resolution}" \
    '["incident","resolved"]'
}

# ============================================================
# Example usage
# ============================================================
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
  echo "Example: Creating deployment annotation"

  ANNOTATION_ID=$(annotate_deployment \
    "payments-api" \
    "v1.2.3" \
    "production" \
    "abc123def456")

  echo "Created: ${ANNOTATION_ID}"
fi
```

---

<a name="types"></a>
## 9. 📌 Annotation Types & Use Cases

### Common Annotation Patterns

```
Deployment Events
  Tags:     deployment, <service>, <version>, <environment>
  Text:     "Deployed payments-api v1.2.3 to production"
  Type:     Point
  When:     During CI/CD pipeline after successful deploy

Incident Events
  Tags:     incident, <incident-id>, <severity>, <service>
  Text:     "[SEV-1] INC-0234: Payment processing down"
  Type:     Region (open on start, close on resolution)
  When:     When incident is created/resolved in IRM

Maintenance Windows
  Tags:     maintenance, <service>, planned
  Text:     "Scheduled maintenance: database upgrade"
  Type:     Region (pre-scheduled)
  When:     Before maintenance window begins

Configuration Changes
  Tags:     config-change, <service>, <environment>
  Text:     "Updated connection pool limit: 100 → 200"
  Type:     Point
  When:     When infrastructure config is modified

Scaling Events
  Tags:     scaling, <service>, <direction>
  Text:     "Auto-scaled payments-api: 3 → 8 replicas"
  Type:     Point
  When:     When HPA triggers a scaling event

Feature Flags
  Tags:     feature-flag, <flag-name>, <state>
  Text:     "Enabled feature flag: new-checkout-flow"
  Type:     Point
  When:     When feature flags are toggled

Traffic Events
  Tags:     traffic, <event-type>
  Text:     "Black Friday traffic spike expected"
  Type:     Region
  When:     During known high-traffic periods
```

---

<a name="bestpractices"></a>
## 10. ✅ Best Practices

```
Tag Design:
✅ Always include the service name as a tag
✅ Always include the environment as a tag
✅ Use consistent lowercase tags
✅ Include the event type as the first tag
   (deployment, incident, maintenance)
✅ Include version numbers for deployments
✅ Include incident IDs for incident annotations

Annotation Management:
✅ Clean up old annotations periodically
   Use the DELETE API for stale annotations
✅ Use region annotations for time-spanning events
   (incidents, maintenance windows)
✅ Update annotations rather than creating duplicates
   (use PATCH to update on resolution)
✅ Limit annotations per time range
   Too many annotations make dashboards unreadable
✅ Use a service account with minimum permissions
   Annotation Writer role is sufficient

Dashboard Configuration:
✅ Enable the built-in Grafana annotation datasource
   on dashboards that should show annotations
✅ Create annotation queries filtered by relevant tags
   (e.g. show only annotations tagged with the service)
✅ Test annotation visibility in the dashboard UI
   after creating programmatically

Security:
✅ Use service account tokens, not personal tokens
✅ Use the minimum required role
   (Annotation Writer does not need admin)
✅ Store tokens in secrets manager
✅ Rotate tokens every 90 days
✅ Never log token values
```

---

<a name="faqs"></a>
## 11. ❓ Frequently Asked Questions

---

**Q: What is the minimum permission required to create annotations?**

> The service account needs the **Annotation Writer** role or a custom role with `annotations:create` and `annotations:write` permissions. It does not need Admin or Editor role just for annotations.

---

**Q: How do I make annotations appear on all panels of a dashboard?**

> Omit the `panelId` field from the API payload. The annotation will appear across all panels. To target a specific panel, include `"panelId": <id>`.

---

**Q: How do I get a dashboard UID?**

> Navigate to the dashboard in Grafana. The UID is in the URL: `https://grafana.com/d/<UID>/dashboard-name`. You can also get it via the API: `GET /api/dashboards/db/<slug>` or `GET /api/search?query=<name>`.

---

**Q: Can annotations be created in bulk?**

> Yes — call the `POST /api/annotations` endpoint in a loop. There is no bulk endpoint, but the API is fast enough to handle hundreds of annotations sequentially. Add a small delay between requests to avoid rate limiting.

---

**Q: How do I show annotations from the API on my dashboard?**

> In the dashboard settings, go to **Annotations** → **Add annotation query** → select **Grafana** as the datasource → filter by tags. This will display all annotations matching those tags on the dashboard timeline.

---

## 📚 Reference Resources

| Resource | Location |
|----------|----------|
| Grafana Annotations API | `grafana.com/docs/grafana/latest/developers/http_api/annotations` |
| Grafana Python Client | `github.com/grafana-toolbox/grafana-client` |
| Grafana Go OpenAPI Client | `github.com/grafana/grafana-openapi-client-go` |
| Grafana Terraform Provider | `registry.terraform.io/providers/grafana/grafana` |
| Grafana Service Accounts | `grafana.com/docs/grafana/latest/administration/service-accounts` |
