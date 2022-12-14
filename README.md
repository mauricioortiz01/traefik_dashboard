# Exposing Traefik Dashboard

1. Create an EC2 instance
1. Install kubectl
  ```
  curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.7/2022-10-31/bin/linux/amd64/kubectl
  chmod +x ./kubectl
  mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
  echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
  kubectl version --short --client
  ```
1. Update AWS CLI version to V2
  ```
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && sudo ./aws/install
  ```
1. Configure AWS account using `aws configure`
1. Create EKS cluster `./eks_startup.sh`
1. Install Helm
  ```
  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  chmod 700 get_helm.sh
  ./get_helm.sh
  ```
1. Add Traefik helm chart `helm repo add traefik https://helm.traefik.io/traefik`
1. Store Traefik config in a file called `config.yaml` with the following contents
  ```
  additionalArguments:
  - --entrypoints.websecure.http.tls.domains[0].main=*.com
  - --entrypoints.websecure.http.tls.domains[0].sans=*.com
  - --accesslog=true
  - --tracing=false

  ingressRoute:
    dashboard:
      enabled: false

  logs:
    general:
      level: DEBUG

  ## Static configuration
  serversTransport:
    insecureSkipVerify: true

  service:
    enabled: true
    type: LoadBalancer
    # Additional annotations applied to both TCP and UDP services (e.g. for cloud provider specific config)
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-internal: "false"
  ```
1. Install Traefik chart with custom config values `helm install -f config.yaml traefik traefik/traefik`
1. Wait for Loadbalancer to come up on AWS and copy DNS
1. Create `traefik.yaml` and add Loadbalancer DNS 
  ```
  apiVersion: traefik.containo.us/v1alpha1
  kind: IngressRoute
  metadata:
    name: traefik-dashboard
  spec:
    entryPoints:
      - web
    routes:
      - match: Host(`ad86b5bfb408c459f83ebad829f9769b-e55249426b7df615.elb.us-east-2.amazonaws.com`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
        kind: Rule
        services:
          - name: api@internal
            kind: TraefikService
  ```
1. Apply IngressRoute `kubectl apply -f traefik.yaml`
1. Visit AWS Loadbalancer DNS `LOADBALANCERDNS.COM`
 
