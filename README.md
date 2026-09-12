# Scenario 1 — Service Catalog Debugging and Fix

## Problem Statement

The `service-catalog` application was not working correctly.

The expected request flow was:

```text
Client
   |
   v
Nginx
   |
   v
Backend
   |
   v
PostgreSQL
```

The required validation was:

```bash
curl http://localhost/graph
```

The endpoint was expected to return HTTP `200` with the service catalog data.

Initially, the endpoint returned:

```text
HTTP/1.1 502 Bad Gateway
```

---

## Investigation

### 1. Host DNS Resolution

The first issue encountered during troubleshooting was DNS resolution on the VM.

`/etc/resolv.conf` was pointing to the local systemd-resolved stub resolver:

```text
nameserver 127.0.0.1
```

I checked the system resolver configuration and found that `systemd-resolved` was not providing the expected DNS resolution.

I fixed the resolver configuration by restarting/enabling `systemd-resolved` and changing `/etc/resolv.conf` to use:

```text
/run/systemd/resolve/resolv.conf
```

After the change, DNS resolution on the host was working correctly.

---

## 2. Initial Application Check

After resolving the host DNS issue, I started the application using Docker Compose and tested the required endpoint:

```bash
curl -i http://localhost/graph
```

The result was:

```text
HTTP/1.1 502 Bad Gateway
```

Therefore, I continued the investigation through the container logs.

---

## 3. Backend Database Connectivity

I checked the backend container logs.

The backend reported:

```text
psycopg2.OperationalError:
could not translate host name "db" to address:
Name or service not known
```

This indicated that the backend could not resolve the PostgreSQL service named `db`.

I inspected the Docker networks and found:

```text
backend
  -> nginx-backend-net

db
  -> backend-db-net
```

The backend and database did not share a common Docker network.

### Root Cause

The `backend` service could not resolve the `db` service because they were attached to different Docker networks.

---

## 4. Docker Network Fix

I updated `docker-compose.yml` so that the backend is connected to both required networks:

```yaml
backend:
  networks:
    - nginx-backend-net
    - backend-db-net
```

The resulting network topology is:

```text
                  nginx-backend-net
                 /                 \
              Nginx              Backend
                                   |
                                   |
                           backend-db-net
                                   |
                                   |
                                  DB
```

After recreating the containers, the backend was able to resolve the `db` service.

During the initial startup, PostgreSQL also needed time to become ready to accept connections. The backend startup script already retries the database connection before starting Gunicorn, so the backend eventually connected successfully.

The backend logs then showed successful application startup and Gunicorn listening on:

```text
0.0.0.0:5000
```

---

## 5. Nginx Upstream Configuration

After fixing the backend-to-database connectivity, the application was still returning:

```text
HTTP/1.1 502 Bad Gateway
```

I checked the Nginx error logs.

Nginx reported:

```text
backend-api could not be resolved (3: Host not found)
```

I inspected the Nginx configuration and found that the upstream was configured as:

```nginx
http://backend-api:8080
```

However, the Docker Compose service was named:

```text
backend
```

and the application was listening on:

```text
5000
```

### Root Cause

Nginx was configured to connect to a non-existent Docker service name and the wrong application port.

---

## 6. Nginx Fix

I changed the Nginx upstream configuration from:

```nginx
http://backend-api:8080
```

to:

```nginx
set $backend_upstream http://backend:5000;
```

This matches the Docker Compose service name and the port exposed by the Flask/Gunicorn backend.

After applying the configuration, I restarted/recreated the Nginx container.

---

## 7. Final Validation

I tested the required endpoint again:

```bash
curl -i http://localhost/graph
```

The final response was:

```text
HTTP/1.1 200 OK
Server: nginx/1.27.5
Content-Type: application/json
```

The response contained the expected service catalog data, including the nodes and edges.

Example:

```json
{
  "edges": [
    {
      "from": "frontend",
      "id": 1,
      "to": "backend"
    }
  ],
  "nodes": [
    {
      "id": 1,
      "kind": "service",
      "name": "frontend",
      "type": "frontend"
    }
  ]
}
```

Therefore, the complete request path was successfully restored:

```text
Client
   |
   | HTTP :80
   v
Nginx
   |
   | backend:5000
   v
Backend
   |
   | db:5432
   v
PostgreSQL
```

## Final Result

The `service-catalog` application was successfully restored.

The final health check:

```bash
curl http://localhost/graph
```

returned:

```text
HTTP 200 OK
```

with the expected service catalog JSON response.

### Root Causes Found

1. Host DNS resolver configuration was incorrect.
2. Backend and PostgreSQL were not connected to a common Docker network.
3. Nginx was configured with an incorrect backend service name and port.

### Fixes Applied

1. Fixed the host DNS resolver configuration.
2. Connected `backend` to `backend-db-net`.
3. Changed the Nginx upstream from `backend-api:8080` to `backend:5000`.
4. Verified the application through the required `/graph` endpoint.
