# DevOps with ArgoCD and GitHub Actions

This is a practical example of how to configure ArgoCD, GitHub Actions, and deploy applications using Kubernetes. This project is designed to demonstrate the key aspects of a DevOps engineer's workflow.

## Project Overview

This project includes:

1. A simple Python server that listens on port 80 and responds with status code 200 on /.
2. A Dockerfile to containerize the Python server.
3. GitHub Actions workflow to build and push the Docker image to Docker Hub.
4. Terraform code to create an EKS cluster, deploy ArgoCD using Helm, and configure a Route53 DNS record.
5. A Kubernetes manifest to deploy the app using ArgoCD.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
  - [Step 1: Python Server and Dockerfile](#step-1)
  - [Step 2: GitHub Actions Workflow](#step-2)
  - [Step 3: Terraform Setup for EKS and ArgoCD](#step-3)
  - [Step 4: ArgoCD Application](#step-4)
- [Result](#result)
- [Conclusion](#conclusion)
- [Contributing](#contributing)
- [License](#license)

## <a name="prerequisites"></a>Prerequisites

- AWS CLI configured with the necessary permissions.
- Docker and Docker Hub account.
- Terraform installed.
- Kubernetes CLI (kubectl) installed.
- GitHub repository setup with necessary secrets.

## <a name="setup-guide"></a>Setup Guide

### <a name="step-1"></a>Step 1: Python Server and Dockerfile

The Python server is a simple HTTP server implemented using Python's http.server module. The server listens on port 80 and responds with a simple HTML page.

#### Python Server Code (server.py):

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

hostName = "0.0.0.0"
serverPort = 80

class MyServer(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/stress":
            self.simulate_load()

        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()

        if self.path == "/":
            self.wfile.write(bytes("<html><head><title>STEP 4</title>", "utf-8"))
            self.wfile.write(bytes("<style>", "utf-8"))
            self.wfile.write(bytes("body {background-color: black; color: white; height: 100vh; display: flex; justify-content: center; align-items: center; margin: 0; font-family: 'Montserrat', sans-serif;}", "utf-8"))
            self.wfile.write(bytes("</style></head>", "utf-8"))
            self.wfile.write(bytes("<body>", "utf-8"))
            self.wfile.write(bytes("<h1>STEP 4 UPDATE</h1>", "utf-8"))
            self.wfile.write(bytes("</body></html>", "utf-8"))
        elif self.path == "/stress":
            self.wfile.write(bytes("<html><head><title>Stress Test</title></head>", "utf-8"))
            self.wfile.write(bytes("<body><h1>Stress Test Page</h1></body></html>", "utf-8"))

    def simulate_load(self):
        def fib(n):
            if n < 2:
                return n
            return fib(n - 1) + fib(n - 2)
        fib(32)

if __name__ == "__main__":
    webServer = HTTPServer((hostName, serverPort), MyServer)
    print("Server started http://%s:%s" % (hostName, serverPort))
    try:
        webServer.serve_forever()
    except KeyboardInterrupt:
        pass
    webServer.server_close()
    print("Server stopped.")

```

#### Dockerfile:

```Dockerfile
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip

WORKDIR /app
COPY . /app

CMD ["python3", "server.py"]
```

### <a name="step-2"></a>Step 2: GitHub Actions Workflow

The GitHub Actions workflow builds and pushes the Docker image to Docker Hub upon every push to the main branch.

#### GitHub Actions Workflow (`.github/workflows/build.yml`):

```yaml
name: action-1

on:
  push:
    branches:
      - main
    paths:
      - "server.py"
      - "Dockerfile"
  pull_request:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v1

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Log in to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: wincchesster/step-4-app:latest, wincchesster/step-4-app:${{ github.sha }}

      - name: Test
        run: |
          docker run --rm -d -p 8080:80 --name test-container wincchesster/step-4-app:latest
          sleep 10 
          response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080)
          if [ "$response" -ne 200 ]; then
            echo "Test Failed: HTTP response is $response"
            exit 1
          fi

      - name: Cleanup
        run: docker rm -f test-container

      - name: Get the current commit hash
        id: vars
        run: echo "::set-output name=hash::$(git rev-parse HEAD)"

      - name: Update deployment file with the new commit hash
        run: |
          sed -i 's/commit-hash: .*/commit-hash: "'${{ steps.vars.outputs.hash }}'"/' dev/deployment.yml

      - name: Commit and push changes
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add deploy/deployment.yaml
          git commit -m "Update deployment with commit hash"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### <a name="step-3"></a>Step 3: Terraform Setup for EKS and ArgoCD

Use Terraform to create an EKS cluster, install ArgoCD using Helm, and set up Route53 DNS records.

#### Terraform Code:

```hcl
resource "kubernetes_namespace" "argocd" {
  metadata {
    name = "argocd"
  }
}

resource "helm_release" "argocd" {
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  namespace  = kubernetes_namespace.argocd.metadata[0].name

  values = [
    <<EOF
server:
  service:
    type: LoadBalancer
  extraArgs:
    - --insecure
EOF
  ]
}

data "kubernetes_service" "argocd_server" {
  metadata {
    name      = "argocd-server"
    namespace = kubernetes_namespace.argocd.metadata[0].name
  }

  depends_on = [helm_release.argocd]
}

resource "aws_route53_record" "argocd" {
  zone_id = data.aws_route53_zone.selected.zone_id
  name    = "argocd.${var.name}"
  type    = "CNAME"
  ttl     = 300

  records = [
    data.kubernetes_service.argocd_server.status[0].load_balancer[0].ingress[0].hostname
  ]

  depends_on = [helm_release.argocd]
}
```

### <a name="step-4"></a>Step 4: ArgoCD Application

Create a Kubernetes manifest to deploy the application using ArgoCD.

#### ArgoCD Application Manifest (`dev/deployment.yml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: step-4-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: "https://github.com/wincchesster/step-4-app.git"
    targetRevision: HEAD
    path: dev
  destination:
    server: "https://kubernetes.default.svc"
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## <a name="result"></a>Result

1. Python Server: A simple HTTP server running on port 80.
2. Docker Image: Built and pushed to Docker Hub using GitHub Actions.
3. EKS Cluster: Created using Terraform.
4. ArgoCD: Installed in the EKS cluster using Helm.
5. Application Deployment: Automatically updated in the EKS cluster using ArgoCD.

## <a name="conclusion"></a>Conclusion

This project demonstrates the automation of the CI/CD pipeline using GitHub Actions and ArgoCD. It also provides a practical example of deploying applications to Kubernetes using modern DevOps practices.

## <a name="contributing"></a>Contributing

Contributions are welcome! Please fork the repository and submit a pull request for review.