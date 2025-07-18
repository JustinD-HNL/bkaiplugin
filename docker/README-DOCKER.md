# Docker Environment for AI Error Analysis Buildkite Plugin

This Docker setup provides a fully functional Buildkite agent with the AI Error Analysis plugin pre-installed for testing, development, and production deployment.

## Quick Start

### 1. Build the container
```bash
# From the project root
./docker/docker-build.sh

# Or from the docker directory
cd docker
./docker-build.sh
```

### 2. Set up environment variables
Copy the example environment file and add your API keys:
```bash
cd docker
cp .env.example .env
# Edit .env and add your BUILDKITE_AGENT_TOKEN and AI provider API key
```

### 3. Run the container

**Interactive shell (for testing and development):**
```bash
# From the project root
./docker/docker-run.sh
```

**Start Buildkite agent:**
```bash
./docker/docker-run.sh agent --token YOUR_BUILDKITE_AGENT_TOKEN
```

**Run plugin tests:**
```bash
./docker/docker-run.sh test
```

## Container Features

- **Base OS**: Ubuntu 24.04
- **Python**: 3.12 with virtual environment
- **Buildkite Agent**: Latest version with plugin pre-installed
- **AI Providers**: Support for OpenAI, Anthropic, and Google Gemini
- **Development Tools**: git, curl, jq, vim

## Environment Variables

### Required for Agent Mode
- `BUILDKITE_AGENT_TOKEN`: Your Buildkite agent token

### AI Provider Configuration (set at least one)
- `OPENAI_API_KEY`: For OpenAI GPT models
- `ANTHROPIC_API_KEY`: For Anthropic Claude models
- `GOOGLE_API_KEY`: For Google Gemini models

### Plugin Configuration
- `AI_PROVIDER`: Choose provider (openai, anthropic, google)
- `AI_MODEL`: Specify model (e.g., gpt-4o, claude-3-opus-20240229, gemini-1.5-pro)
- `BUILDKITE_PLUGIN_AI_ERROR_ANALYSIS_DEBUG`: Enable debug logging (true/false)

## Usage Examples

### 1. Test the plugin locally
```bash
# Start container with OpenAI (from project root)
OPENAI_API_KEY=sk-... ./docker/docker-run.sh

# Inside container, test the plugin
cd test-pipeline
buildkite-agent pipeline upload
```

### 2. Run with docker-compose
```bash
# From the docker directory
cd docker

# Start services
docker-compose up

# Or run specific commands
docker-compose run --rm buildkite-agent agent
docker-compose run --rm buildkite-agent test
```

### 3. Test with different AI providers
```bash
# OpenAI (from project root)
AI_PROVIDER=openai AI_MODEL=gpt-4o ./docker/docker-run.sh

# Anthropic
AI_PROVIDER=anthropic AI_MODEL=claude-3-opus-20240229 ./docker/docker-run.sh

# Google
AI_PROVIDER=google AI_MODEL=gemini-1.5-pro ./docker/docker-run.sh
```

## Testing the Plugin

The container includes a test pipeline at `/home/buildkite/test-pipeline/pipeline.yml` that intentionally fails to trigger the AI error analysis:

```yaml
steps:
  - label: ":hammer: Test Build with Error"
    command: |
      echo "Starting build..."
      echo "This will fail to trigger the AI error analysis"
      exit 1
    plugins:
      - ai-error-analysis:
          enabled: true
          provider: "${AI_PROVIDER:-openai}"
          model: "${AI_MODEL:-gpt-4o}"
```

## Development Workflow

1. The plugin source is mounted as a volume, so changes are reflected immediately
2. Python dependencies are installed in a virtual environment at `/home/buildkite/.venv`
3. The plugin is symlinked to the agent's plugin directory
4. Use `docker-compose logs` to view agent output

## Troubleshooting

### Container won't start
- Check Docker is running: `docker ps`
- Verify build completed: `docker images | grep buildkite`

### Agent won't connect
- Verify `BUILDKITE_AGENT_TOKEN` is set correctly
- Check network connectivity from container
- Review agent logs: `cd docker && docker-compose logs buildkite-agent`

### Plugin not working
- Ensure AI provider API key is set
- Check plugin is enabled in pipeline YAML
- Enable debug mode: `BUILDKITE_PLUGIN_AI_ERROR_ANALYSIS_DEBUG=true`
- Review Python environment: `cd docker && docker-compose run --rm buildkite-agent bash -c "which python && python --version"`

### Permission issues
- The container runs as the `buildkite` user (non-root)
- Plugin files are mounted read-only by default
- Agent data is persisted in a Docker volume

## Deployment Options

### Using the deploy.sh Script

The `deploy.sh` script provides multiple deployment options:

```bash
# Build the image
./docker/deploy.sh build

# Run locally with docker-compose
./docker/deploy.sh compose

# Run standalone container
BUILDKITE_AGENT_TOKEN=xxx OPENAI_API_KEY=sk-xxx ./docker/deploy.sh run

# Push to Docker registry
REGISTRY_URL=docker.io/username ./docker/deploy.sh push

# Export image for offline deployment
./docker/deploy.sh export my-plugin.tar
```

### Deploy to Docker Registry

1. **Docker Hub:**
```bash
REGISTRY_URL=docker.io/yourusername IMAGE_TAG=v1.0.0 ./docker/deploy.sh push
```

2. **AWS ECR:**
```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Push image
REGISTRY_URL=123456789.dkr.ecr.us-east-1.amazonaws.com IMAGE_TAG=latest ./docker/deploy.sh push
```

3. **Google Container Registry:**
```bash
# Configure Docker for GCR
gcloud auth configure-docker

# Push image
REGISTRY_URL=gcr.io/your-project IMAGE_TAG=latest ./docker/deploy.sh push
```

### Deploy to Kubernetes

1. **Update the image in kubernetes-deployment.yaml:**
```yaml
image: your-registry.com/ai-error-analysis-buildkite-plugin:latest
```

2. **Create secrets:**
```bash
kubectl create secret generic buildkite-agent-secrets \
  --from-literal=agent-token=YOUR_BUILDKITE_TOKEN \
  --from-literal=openai-api-key=YOUR_OPENAI_KEY \
  -n buildkite
```

3. **Apply the deployment:**
```bash
kubectl apply -f docker/kubernetes-deployment.yaml
```

4. **Check deployment status:**
```bash
kubectl get pods -n buildkite
kubectl logs -f deployment/buildkite-ai-error-agent -n buildkite
```

### Deploy with Docker Swarm

```bash
# Initialize swarm (if not already done)
docker swarm init

# Create secrets
echo "YOUR_BUILDKITE_TOKEN" | docker secret create buildkite-token -
echo "YOUR_OPENAI_KEY" | docker secret create openai-key -

# Deploy stack
docker stack deploy -c docker/docker-compose.yml buildkite-ai
```

### Production Considerations

1. **Security:**
   - Never hardcode API keys in images
   - Use secrets management (Kubernetes Secrets, Docker Secrets, AWS SSM, etc.)
   - Scan images for vulnerabilities: `docker scan ai-error-analysis-buildkite-plugin:latest`

2. **Monitoring:**
   - Enable agent metrics endpoint
   - Use log aggregation (ELK, CloudWatch, etc.)
   - Set up alerts for agent disconnections

3. **Scaling:**
   - Adjust replica count based on workload
   - Use horizontal pod autoscaling in Kubernetes
   - Consider node affinity for GPU workloads

4. **Updates:**
   - Use semantic versioning for images
   - Implement rolling updates
   - Test in staging environment first