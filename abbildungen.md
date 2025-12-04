Abbildung 1

```yaml
services:
  <service-name>:
    image: repo/image:tag
    ports:
      - "8080:80"
    environment:
      - KEY=value
    volumes:
      - ./data:/app/data
    depends_on:
      - other-service
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - appnet
```

Abbildung 2
```yaml
keycloak:
  image: keycloak/keycloak:26.4.2
  environment:
    KC_DB: postgres
    KC_DB_URL: jdbc:postgresql://keycloak-database:5432/keycloak
    KC_DB_USERNAME: keycloak
    KC_DB_PASSWORD: keycloak_password
    KEYCLOAK_ADMIN: admin
    KEYCLOAK_ADMIN_PASSWORD: admin
    KC_HOSTNAME_STRICT: false
    KC_HOSTNAME_STRICT_HTTPS: false
    KC_HTTP_ENABLED: true
  command:
    - start-dev
  ports:
    - "8080:8080"
  depends_on:
    keycloak-database:
      condition: service_healthy
  networks:
    default:
      aliases:
        - keycloak.localhost.pomerium.io
```

Abbildung 3 
```yaml
keycloak-database:
  image: postgres:15
  environment:
    POSTGRES_DB: keycloak
    POSTGRES_USER: keycloak
    POSTGRES_PASSWORD: keycloak_password
  volumes:
    - postgres_data:/var/lib/postgresql/data
  healthcheck:
    test: [ "CMD", "pg_isready", "-U", "keycloak" ]
    interval: 10s
    timeout: 5s
    retries: 5
```

Abbildung 4
```yaml
volumes:
  postgres_data:
    name: keycloak_postgres_data
  grafana_data:
    name: grafana_data
```

Abbildung 5
```yaml
pomerium:
  image: pomerium/pomerium:v0.31.1
  volumes:
    - ./config/pomerium-config.yaml:/pomerium/config.yaml:ro
  ports:
    - 443:443
```

Abbildung 6
```yaml
idp_provider: oidc
idp_client_id: 'grafana_client'
idp_client_secret: 'client_secret'
idp_provider_url: 'http://keycloak.localhost.pomerium.io:8080/realms/grafana_realm'
```

Abbildung 7
```yaml
grafana:
  image: grafana/grafana:12.2.1
  volumes:
    - ./config/grafana.ini:/etc/grafana/grafana.ini:ro
    - grafana_data:/var/lib/grafana
    - ./config/jwks.json:/etc/grafana/jwks/jwks.json:ro
  depends_on:
    - keycloak
    - pomerium
```

Abbildung 8
```
[server]
root_url = https://grafana.localhost.pomerium.io
serve_from_sub_path = true

[auth]
disable_login_form = true
signout_redirect_url = https://grafana.localhost.pomerium.io/.pomerium/sign_out

[auth.jwt]
enabled = true
header_name = X-Pomerium-Jwt-Assertion
email_claim = sub
username_claim = sub
jwk_set_file = /etc/grafana/jwks/jwks.json
auto_sign_up = true
role_attribute_path = contains(roles, 'admin') && 'Admin' || contains(roles, 'editor') && 'Editor' || 'Viewer'
allow_assign_grafana_admin = true
skip_org_role_sync = false

[users]
auto_assign_org = true
```

Abbildung 11
```json
[
  {
    "level": "info",
    "server-name": "all",
    "service": "authorize",
    "request-id": "e77f1d33-c8e0-4bbd-97a1-71bb19f9f8ec",
    "check-request-id": "e77f1d33-c8e0-4bbd-97a1-71bb19f9f8ec",
    "method": "GET",
    "path": "/",
    "host": "grafana.localhost.pomerium.io",
    "ip": "172.18.0.1",
    "session-id": "0d58b069-7723-a9cd-c76f-839a97310f42",
    "user": "b847e911-a80b-43fc-bf03-c33ca67fece8",
    "email": "admin@interne-firma.xyz",
    "envoy-route-checksum": 17832595174180556727,
    "envoy-route-id": "2317b78b316a1d67",
    "route-checksum": 17832595174180556727,
    "route-id": "",
    "allow": true,
    "allow-why-true": [
      "domain-ok"
    ],
    "deny": false,
    "deny-why-false": [],
    "time": "2025-11-14T13:04:54Z",
    "message": "authorize check"
  },
  {
    "level": "info",
    "server-name": "all",
    "service": "envoy",
    "upstream-cluster": "route-2317b78b316a1d67",
    "method": "GET",
    "authority": "grafana:3000",
    "path": "/",
    "user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:144.0) Gecko/20100101 Firefox/144.0",
    "referer": "",
    "forwarded-for": "172.18.0.1",
    "request-id": "e77f1d33-c8e0-4bbd-97a1-71bb19f9f8ec",
    "duration": 30.98425,
    "size": 64812,
    "response-code": 200,
    "response-code-details": "via_upstream",
    "time": "2025-11-14T13:04:55Z",
    "message": "http-request"
  }
]
```
Abbildung 12
```yaml
  loki:
    image: grafana/loki:3.5.8
    command: [ "-config.file=/etc/loki/config.yml" ]
    volumes:
      - ./config/loki-config.yml:/etc/loki/config.yml:ro
      - ./.loki-data:/loki
    restart: unless-stopped
```

Abbildung 13
```yaml
server:
  http_listen_port: 3100
```

Abbildung 14
```yaml
  alloy:
    image: grafana/alloy:v1.11.0
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config/alloy-config.alloy:/etc/alloy/config.alloy:ro
    depends_on:
      - loki
    restart: unless-stopped
```

Abbildung 15
```terraform
discovery.docker "docker" {
	host             = "unix:///var/run/docker.sock"
	refresh_interval = "5s"
}

loki.process "docker" {
	forward_to = [loki.write.default.receiver]

	stage.json {
		expressions = {
			level   = "level",
			message = "message",
			name    = "name",
			service = "service",
			time    = "time",
		}
	}

	stage.labels {
		values = {
			level   = null,
			name    = null,
			service = null,
		}
	}

	stage.structured_metadata {
      values = {
        path   = "",
        reason = "",
      }
    }
}

discovery.relabel "docker" {
	targets = []

	rule {
		source_labels = ["__meta_docker_container_name"]
		regex         = "/(.*)"
		target_label  = "container"
	}

	rule {
		source_labels = ["__meta_docker_container_label_promtail_job"]
		target_label  = "job"
	}

	rule {
		source_labels = ["__meta_docker_container_label_com_docker_compose_service"]
		target_label  = "service"
	}
}

loki.source.docker "docker" {
	host             = "unix:///var/run/docker.sock"
	targets          = discovery.docker.docker.targets
	forward_to       = [loki.process.docker.receiver]
	relabel_rules    = discovery.relabel.docker.rules
	refresh_interval = "5s"
}

loki.write "default" {
	endpoint {
		url = "http://loki:3100/loki/api/v1/push"
	}
	external_labels = {}
}
```

abbildung 16
```yaml
- from: https://grafana.localhost.pomerium.io
  to: http://grafana:3000
  path: /admin
  policy:
    - allow:
        and:
          - domain:
              is: interne-firma.xyz
          - claim/roles: "admin"
  allow_any_authenticated_user: false
  pass_identity_headers: true
  set_request_headers:
    X-Forwarded-Proto: https
```

abbildung 18
```yaml
- from: https://grafana.localhost.pomerium.io
  to: http://grafana:3000
  policy:
    allow:
      or:
        - domain:
            is: interne-firma.xyz
        - domain:
            is: externe-firma.xyz
  allow_any_authenticated_user: false
  pass_identity_headers: true
  set_request_headers:
    X-Forwarded-Proto: https
```

abbildung 19
```json
{
  "level": "info",
  "server-name": "all",
  "service": "authorize",
  "request-id": "1c3c1571-cb35-4e23-966d-cb0c5e6e83c0",
  "check-request-id": "1c3c1571-cb35-4e23-966d-cb0c5e6e83c0",
  "method": "GET",
  "path": "/admin",
  "host": "grafana.localhost.pomerium.io",
  "ip": "172.18.0.1",
  "session-id": "97c7c7b8-0dd5-554d-e534-b076b0ed86e1",
  "user": "313b6570-1320-4c4f-b3f6-5f914fd11dda",
  "email": "viewer@externe-firma.xyz",
  "envoy-route-checksum": 2170751405052415627,
  "envoy-route-id": "7eaac6f90a75564f",
  "route-checksum": 2170751405052415627,
  "route-id": "",
  "allow": false,
  "allow-why-false": [
    "claim-unauthorized",
    "domain-unauthorized"
  ],
  "deny": false,
  "deny-why-false": [],
  "time": "2025-11-20T10:22:41Z",
  "message": "authorize check"
}
```

abbildung 16
```yaml
```