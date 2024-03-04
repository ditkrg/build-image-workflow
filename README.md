# Build Image Workflow

This GitHub Action automates the process of building and pushing a Docker image to a self-hosted registry. The workflow includes setting up Docker Buildx, extracting metadata, logging in to the registry, and building and pushing the Docker image.

## Inputs

### `image` (required)
- Description: Image Name
- Required: true

### `build-args` (optional)
- Description: Build Arguments
- Required: false

### `file` (optional)
- Description: Dockerfile Path
- Required: false

### `registry` (required)
- Description: Registry URL
- Required: true
- Default: reg.dev.krd

### `username` (required)
- Description: Username for the registry
- Required: true

### `password` (required)
- Description: Password for the registry
- Required: true

### `build-secrets` (optional)
- Description: Build Secrets
- Required: false

## Outputs

### `tag`
- Description: Image Tag
- Value: ${{ steps.meta.outputs.tags[0] }}

### `tags`
- Description: Image Tags
- Value: ${{ steps.meta.outputs.tags }}

## Workflow Steps

1. **Set up Docker Buildx:**
   - Uses: docker/setup-buildx-action@v3

2. **Extract Metadata:**
   - Uses: docker/metadata-action@v5
   - Inputs:
     - `images`: ${{ inputs.registry }}/${{ inputs.image }}
     - `flavor`: latest=false
     - `tags`:
       - Cache: `type=raw,value=${{ github.ref_name }}-cache`
       - Branches: `type=ref,event=branch`, `type=ref,event=branch,suffix=-{{sha}},priority=8888`
       - Releases: `type=semver,pattern={{major}}`, `type=semver,pattern={{major}}.{{minor}}`, `type=semver,pattern={{version}},priority=9999`

3. **Login to Registry:**
   - Uses: docker/login-action@v3
   - Inputs:
     - `registry`: ${{ inputs.registry }}
     - `username`: ${{ inputs.username }}
     - `password`: ${{ inputs.password }}

4. **Build Docker images:**
   - Uses: docker/build-push-action@v5
   - Inputs:
     - `push`: true
     - `file`: ${{ inputs.file }}
     - `tags`: ${{ steps.meta.outputs.tags }}
     - `cache-to`: `type=inline`
     - `cache-from`: `type=registry,ref=${{ inputs.registry }}/${{ inputs.image }}:${{ github.ref_name }}-cache`
     - `build-args`: ${{ inputs.build-args }}
     - `secrets`: ${{ inputs.build-secrets }}

## Example Usage

```yaml
name: Build Image Workflow
on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build and Push Image
        uses: ditkrg/build-image-workflow@v1
        with:
          image: "my-docker-image"
          registry: "my-registry.example.com"
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
          build-args: "EXAMPLE=123"
          build-secrets: "EXAMPLE=****"
          file: "path/to/Dockerfile"
```

If you want to use it with our GitOps Action:

```yaml
name: Deploy

on:
  push:
    branches:
      - dev
      - main
    tags:
      - v[0-9]+.[0-9]+.[0-9]+

    paths-ignore:
      - "**.md"
      - ".vscode/**"

      - ".github/**"
      - "!.github/workflows/tests-base.yaml"
      - "!.github/workflows/deploy.yaml"

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.build-image.outputs.tag }}
      tags: ${{ steps.build-image.outputs.tags }}
    steps:
      - id: build-image
        name: Build and Push Image
        uses: ditkrg/build-image-workflow@v1
        with:
          image: "my-docker-image"
          registry: "my-registry.example.com"
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
          build-args: "EXAMPLE=123"
          build-secrets: "EXAMPLE=****"
          file: "path/to/Dockerfile"

  update-gitops:
    runs-on: ubuntu-latest
    concurrency: pr-${{ github.ref_name }}
    needs: build
    steps:
      - name: Update gitops
        uses: ditkrg/update-gitops-image@v1
        with:
          owner: ditkrg
          repo: GITOPS_REPO
          app-id: ${{ secrets.APP_ID }}
          image-tag: ${{ needs.build.outputs.tag }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          component-name: REPO_NAME
```
