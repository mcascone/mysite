# mysite
[![Basic deploy](https://github.com/mcascone/mysite/actions/workflows/deployToEC2.yaml/badge.svg?branch=main)](https://github.com/mcascone/mysite/actions/workflows/deployToEC2.yaml)

A very simple static website container for proof-of-life testing.

It'll get more sophisticated over time.


# Build

This is a very simple image build. The `Dockerfile` defines the image.

```
docker build -t mcascone/mysite --platform linux/amd64 .
```

***NOTE: You don't have to build this image to use it!*** It is already hosted on `docker.io`, and running the examples below will automatically pull the image if it doesn't exist on your machine.

If you want to play with the contents of the image, feel free to modify anything in this repo. Just change your build command to something other than `mcascone/mysite`, or at the very least, *please don't push to `mcascone/mysite`*.

> Note 2: the `--platform` switch in the build command above is only required if you are building on an M1 Mac and intend to run the image on anything *other than* an M1 Mac. So it's best to just always leave it on by adding `export DOCKER_DEFAULT_PLATFORM=linux/amd64` to your shell profile.

# Usage

There are any number of ways to deploy this site. That's really the beauty of containers, isn't it?

I'll try to scaffold the techniques by approximate level of complexity.

## Run the container locally with `docker compose`
This really couldn't be easier. In a terminal, run: `docker compose up`

This builds and launches the container on `localhost:8085`. View it in a browser or `curl localhost:8085`

Why `8085`? I wanted to set it to a different port than in the next example, just to avoid collisions. Look in the [docker-compose](docker-compose.yml) file for the port definition: `8085:80`.

To knock the instance down, run `docker compose down`.

## Run the container locally using `docker run`
```
docker run --rm -d -p 8084:80 --name mysite mcascone/mysite
```

This will launch the container and expose it to port `8084` on your local machine. To see it, go to http://localhost:8084/ or `curl localhost:8084`
> There's nothing special about the `8084` port, it's just unlikely to be used by another process on your machine. You can change it if needed.

- The `-d` param runs the container in the background, so you get your shell back after launch.
- The `--rm` param automatically deletes the container when it is stopped.
- `-p` exposes the port as discussed above.
- `--name` gives the running container a name. If omitted, one will be automatically generated.

To stop the container, run `docker stop mysite`

## Deploy to kubernetes: manifest

This repo contains a kubernetes deployment manifest that will stand up all the resources necessary for the site:
1. A new namespace `mysite` to isolate the site from other objects in the cluster
2. A deployment of the container in a pod
3. A Load Balancer service to expose the pod to external traffic.

### Deploying with a manifest
Assuming you have a functional k8s instance, run the following:
```
kubectl apply -f deployment.yaml
```

You should see the output:

```
> kubectl apply -f deployment.yaml 
namespace/mysite created
deployment.apps/mysite created
service/mysite-service created
```

Check your deployment with `K9s` or other UI, or the cli:
```
> k get ns
NAME              STATUS   AGE
default           Active   24h
kube-node-lease   Active   24h
kube-public       Active   24h
kube-system       Active   24h
mysite            Active   2m25s

> k get deployment -n mysite
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
mysite   1/1     1            1           99s

> k get pods -n mysite
NAME                      READY   STATUS    RESTARTS   AGE
mysite-78bdbf88c9-shjs4   1/1     Running   0          59s

> k get svc -n mysite
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP                                                                  PORT(S)          AGE
mysite-service   LoadBalancer   10.100.29.12   ae54b0eb4e7bc4829a75012756f06fc1-1859446648.ca-central-1.elb.amazonaws.com   8080:31399/TCP   74s
```

Note the `EXTERNAL-IP` there. That's your new site!

Copy that URL and tack `:8080` on the end, and you should see your site!

### Note on local `minikube` clusters
`minikube` is a great way to test kubernetes stuff on your local machine. Install it with `homebrew install minikube`, start it with `minikube start`. 

To enable the Load Balancer services to work without any Port Forwarding or Ingress shenanigans, you have to enable the `tunnel` feature: `minikube tunnel`. All Load Balancers should then be accessible at `localhost`. 
> Note that this means there's only one URL to use. If you want to expose multiple services at `localhost`, you'll have to specify unique ports for each one.

## Deploy to kubernetes: helm

This repo also contains a Helm chart in the `helm/` directory. 

Helm charts consolidate all of the details of your deployments into a common format that is easily templated and dynamically modified. It's overkill for this example, but it's good to know how it works.

### Deploying with helm

1. Install `helm`.
2. Create a namespace to deploy to (optional but recommended): `kubectl create ns mysite`
3. Run `helm upgrade --install mysite1 helm -n mysite`

You should see the output:
```
Release "mysite1" does not exist. Installing it now.
NAME: mysite1
LAST DEPLOYED: Fri Jan  6 16:05:16 2023
NAMESPACE: mysite
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace mysite svc -w mysite1'
  export SERVICE_IP=$(kubectl get svc --namespace mysite mysite1 --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo http://$SERVICE_IP:8081
  ```

You can follow the instructions printed above to get the URL to your new site!

Check your helm status:
```
> helm list -n mysite
NAME   	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART       	APP VERSION
mysite1	mysite   	1       	2023-01-06 16:05:16.922625 -0600 CST	deployed	mysite-0.1.0	0.2    
```

## Deploy to EC2 with a GitHub Action

By adding the `.github/workflows` directory, we can add GitHub Actions to automate various tasks in our repo.

We'll use the [Bitovi](bitovi.com) [`Deploy Docker to AWS (EC2)` action](https://github.com/marketplace/actions/deploy-docker-to-aws-ec2) to deploy the site to an AWS EC2 instance with one click.

Inspect the `.github/workflows/deployToEC2.yaml` file. Its presence in the repo enables the Action in the GitHub UI.

- You must set environment variables with your AWS access credentials as described here: https://github.com/marketplace/actions/deploy-docker-to-aws-ec2 

Then in the GitHub UI, push a change to the repo, or run the action.


## Deploy to EKS with a GitHub Action
WIP

## Deploy to LocalStack
WIP

## Deploy to GCP/GKS
WIP

## Deploy to Azure/AKS
WIP