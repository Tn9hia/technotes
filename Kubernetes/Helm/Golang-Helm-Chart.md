# Development: Helm Chart for Golang Projects

**Context:** Guide for packaging Golang applications into Helm charts for CI/CD deployment.

## 1. Optimize Docker Image (Multi-stage Build)
Before creating the chart, ensure your Docker image is optimized. Golang binaries are static, so use multi-stage builds to keep the final image tiny.

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Stage 2: Run
FROM alpine:latest  
# Or scratch if you don't need shell/debugging
WORKDIR /root/
COPY --from=builder /app/main .
EXPOSE 8080
CMD ["./main"]
```

## 2. Initialize Chart
Standard structure:
```bash
helm create my-golang-app
```
**Cleanup:** Remove `templates/tests/` if not used immediately.

## 3. Key Configurations (values.yaml)

Customize `values.yaml` for Golang specifics:

```yaml
image:
  repository: my-registry/my-golang-app
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.0.0"

service:
  type: ClusterIP
  port: 8080 # Match your Go app's port

resources: 
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Application Specific Configs
env:
  LOG_LEVEL: "info"
  DB_HOST: "postgres-service"
```

## 4. Template Adjustments

### Deployment (templates/deployment.yaml)
Inject environment variables and ensure probes are set.

```yaml
containers:
  - name: {{ .Chart.Name }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
    ports:
      - name: http
        containerPort: {{ .Values.service.port }}
        protocol: TCP
    # Inject Env Vars
    env:
      {{- range $key, $val := .Values.env }}
      - name: {{ $key }}
        value: {{ $val | quote }}
      {{- end }}
    # Health Checks (Critical for Zero-Downtime)
    livenessProbe:
      httpGet:
        path: /healthz
        port: http
    readinessProbe:
      httpGet:
        path: /ready
        port: http
    resources:
      {{- toYaml .Values.resources | nindent 12 }}
```

## 5. CI/CD Workflow (Example)

Automate linting and packaging in your pipeline (e.g., GitHub Actions, GitLab CI).

1.  **Lint:** Verify chart syntax.
    ```bash
    helm lint ./charts/my-golang-app
    ```
2.  **Package:** Create the `.tgz` artifact.
    ```bash
    helm package ./charts/my-golang-app --version $VERSION --app-version $APP_VERSION
    ```
3.  **Push:** Upload to your registry (Harbor/OCI).
    ```bash
    helm push my-golang-app-$VERSION.tgz oci://registry.example.com/charts
    ```

## 6. Tips for Golang Apps
-   **Graceful Shutdown:** Handle `SIGTERM` in your Go code to finish requests before pod termination.
-   **ConfigMaps:** Use a ConfigMap for non-sensitive config (`config.yaml`) and mount it to `/app/config`.
-   **Secrets:** Use ExternalSecrets or sealed-secrets for DB credentials; avoid plain text in `values.yaml`.
