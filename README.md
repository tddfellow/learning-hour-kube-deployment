# Learning Hour: Deployment of new service onto EKS cluster

In this learning hour, you'll learn how to create and deploy a brand-new service on the EKS cluster.

## Authorizing docker with AWS ECR

AWS ECR is like a Docker Hub, but runs inside your AWS Account. The ECR repository can be configured to be public
or private. For the purposes of this learning hour, we'll use a private repository, since that is what you would
need to use when you're deploying real production services.

```bash
$ aws ecr get-login-password --region {aws_region} | docker login --username AWS --password-stdin {aws_account}.dkr.ecr.{aws_region}.amazonaws.com
```

Where `{aws_region}` and `{aws_account}` are the respective values from your AWS account.

## Building the docker image

_There is an example nodejs express app you can use in the `./app` directory._

```bash
$ docker build -t {aws_account}.dkr.ecr.{aws_region}.amazonaws.com/{repo_name}:vX.Y.Z .
```

Where `{repo_name}` is the name of the ECR repository that already exists in your AWS account.

Where `vX.Y.Z` is the version of the build.

_If you are working through this learning hour in multiple groups,
then let each group choose their own number X (like group 1 = 1, group 2 = 2, etc.)._

## Pushing the docker image to ECR repository

```bash
$ docker push {aws_account}.dkr.ecr.{aws_region}.amazonaws.com/{repo_name}:vX.Y.Z
```

## Creating a deployment manifest

```yaml
# ./manifests/deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  
  # kubernetes namespace that you want to deploy it to
  namespace: {namespace}
  
  # name of the deployment
  name: {name}-deployment

  # label it with the app name
  labels:
    app: {name}

# spec is the definition of this Deployment resource
spec:
  
  # number of instances that you want to have
  replicas: 1

  # apply the following template to the resources by this label  
  selector:
    matchLabels:
      app: {name}

  # template that specifies how each Pod will be configured as part of this Deployment
  template:
    
    # label the template with the name of the app
    metadata:
      labels:
        app: {name}
        
    # the definition of the pod
    spec:
      
      # docker containers that this pod will have
      containers:
        
        # normally, only one container is needed - your server
        - name: server
          
          # docker image that the container will use. 
          # ${VAR} is the syntax to substitute the environment variable
          # in this case the manifest is "parametrized" with BUILD_IMAGE and BUILD_VERSION variables
          image: "${BUILD_IMAGE}:${BUILD_VERSION}"
          
          # whether to try to pull the image at every deployment or not
          imagePullPolicy: Always
          
          # the ports that the container should expose
          ports:
            - containerPort: {port}
              name: http
              protocol: TCP
          
          # environment variables that the container should receive
          env:
            - name: PORT
              value: "{port}"

          # command to execute when starting the container
          command:
            - yarn
            - start
```

Where:
- `{namespace}` is the namespace that you would use to deploy to.
- `{name}` is the name that you want to give to your deployment. This can be `hello-world-{team_number}`.
- `${BUILD_IMAGE}` and ${BUILD_VERSION} are environment variables.
  You'll need to supply further down the line when applying this manifest.
- `{port}` is the port that the container will be exposing.

Labeling the kubernetes resources is important, because it allows later to apply operations (such as deletion, or 
updates) to the whole set of resources at once.

## Creating a Service manifest

A service determines what ports will be exposed from the pod.

```yaml
# ./manifests/service.yml

apiVersion: v1
kind: Service
metadata:
  namespace: {namespace}
  name: {name}-service
  labels:
    app: {name}
spec:
  selector:
    app: {name}
  
  # ports that the container will expose to other kubernetes and AWS resources
  ports:
    - name: http
      port: {port}
```

## Creating an Ingress manifest

An ingress defines how the AWS ALB/ELB resources will connect to the EKS resources (such as the service).

```yaml
# ./manifests/aws-alb-ingress.yml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: {namespace}
  annotations:
    
    # name of the ingress controller
    alb.ingress.kubernetes.io/group.name: ingress-controller
    
    # whether this is internet facing load balancer or internal
    alb.ingress.kubernetes.io/scheme: internet-facing
    
    # which ports should the load balancer listen on
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    
    # type of the target for load balancing (in this case, IP address)
    alb.ingress.kubernetes.io/target-type: ip
    
    # class of ingress = AWS ALB
    kubernetes.io/ingress.class: alb
    
  name: {name}-ingress
spec:
  rules:
    
    # this will be the host that the load balancer will be listening on
    - host: "{subdomain}.${ROUTE53_PUBLIC_ZONE_NAME}"
      http:
        paths:
          
          # define a particular path and its backing service
          # in this case, a single catch-all path is defined
          # however, you can have a single ALB on the same host that routes different URLs to different services
          # (this is what is usually called an API Gateway)
          - backend:
              service:
                
                # the name of the service that this is a target of this load balancer
                name: {name}-service

                # the port that the service is exposing
                port:
                  number: {port}
          
            # path matching rule
            path: /*
            pathType: Prefix
```

Where `{subdomain}` is the subdomain that you want your new service to be available on. This is 
usually your app name, but it doesn't have to be so.

_For this type of Ingress to work, the EKS should have a particular platform application installed that integrates
with the AWS ALB. This is called "Ingress Controller."_

## Kustomize

To put all these files together, you need to create a `kustomize` file:

```yaml
# ./manifests/kustomization.yml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# list of resources to combine
resources:
  - ./deployment.yml
  - ./service.yml
  - ./aws-alb-ingress.yml
```

To apply these manifests (and hence issuing a first deployment), you need to do the following.

First, ensure that you're in the right `kubectl` context and namespace:

```bash
$ kubectl config use-context arn:aws:eks:{aws_region}:{aws_account}:cluster/{cluster_name}
$ kubectl config set-context --current --namespace=learning-hour
```

Second, set the required environment variables and apply the manifests:

```bash
$ export ROUTE53_PUBLIC_ZONE_NAME="{your_basic_environment_domain}"
$ export BUILD_IMAGE="{aws_account}.dkr.ecr.{aws_region}.amazonaws.com/{repo_name}"
$ export BUILD_VERSION="vX.Y.Z"

# preview the result with all variables substituted and all manifests combined:
$ kubectl kustomize | envsubst

# finally, apply
$ kubectl kustomize | envsubst | kubectl apply -f -
```

Where `{your_basic_environment_domain}` is the base domain of your EKS environment.

_PS: you can create a script for this to make it easier to configure all required env vars._

Now you should see your application starting in `$ kubectl get pods`.

Once everything is up, you can hit the `https://{subdomain}.{your_basic_environment_domain}/hello`
and you should see the expected greeting message.

## Adding secrets from AWS Parameter Store

AWS Parameter Store is where encrypted configuration values can be stored and fetched from by various of AWS
resources.

The example application uses one secret environment variable, let's set it using this tool!

```yaml
# ./manifests/secret.yml

apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  name: {name}-secret
  namespace: {namespace}
spec:
  
  # systemManager is the backend that connects to AWS System Manager > Parameter Store
  backendType: systemManager
  data:
    - key: /eks/staging/learning-hour/secret-variable
      name: SECRET_VARIABLE

  template:
    stringData:
      
      # here you can set constant values that are not secrets, they all have to be strings
      SOME_PORT: "12345"

      # you can also use values fetched from the secret storage and interpolate a composed value
      SOME_CONNECTION_URL: |
        protocol://<%= data.SECRET_VARIABLE %>:9876/example
```

And then you have to modify your `deployment.yml` and add the following to the path 
`spec.template.spec.containers.[0]:`

```yaml
# ./manifests/deployment.yml

          # ...
          
          # environment variables that the container should receive
          env:
            - name: PORT
              value: "{port}"
          
          # == ADD THIS ==
          # environment variables that are pulled from the secret
          envFrom:
            - secretRef:
                name: {name}-secret
          # == /END ==

          # ...
```

And don't forget to add the new manifest file to the `kustomization.yml` file:

```yaml
# ./manifests/kustomization.yml

# ...

# list of resources to combine
resources:

  # == ADD THIS ==
  - ./secret.yml
  # == /END ==

  - ./deployment.yml
  # ...
```

Finally, apply the manifests again (don't forget to have the environment variables set):

```
$ kubectl kustomize | envsubst | kubectl apply -f -

# or you can use the script that you've created previously
```

Once everything is applied and pods are restarted, you should be able to hit the 
`https://{subdomin}.{your_basic_environment_domain}/secret` and you should see that the secret
was successfully picked up by your application.

## Bonus: CI automation (optional homework)

You can automate the deployment of this on CI. For this:

1. Create a private repository where you add `app` and `manifests` folders.
2. Create a CI config that has two jobs:
   - build docker image and push it to ECR
   - apply manifests with a concrete version

_You can use `$ git describe --tags` to get the version automatically._
