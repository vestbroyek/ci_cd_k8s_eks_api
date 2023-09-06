# Flask microservice in Kubernetes with EKS and CodePipeline
This repository contains code for creating a Flask API which can be used as a microservice for generating JWT tokens. Alternatively, it can be used as starter code for a more complex API or web application.

The tech stack used is 
- Python 3 with Flask and Gunicorn for the API and pytest for testing
- Docker to containerise the API 
- Kubernetes and EKS for the deployment
- AWS CloudFormation to provision the relevant resources
- AWS CodeBuild and CodePipeline for the CI/CD pipeline

# Table of Contents
# Table of Contents
1. [Flask microservice in Kubernetes with EKS and CodePipeline](#flask-microservice-in-kubernetes-with-eks-and-codepipeline)
2. [API specification](#api-specification)
3. [Demo](#demo)
4. [Walkthrough](#walkthrough)
   - [Prerequisites](#prerequisites)
   - [Testing](#testing)
   - [Deployment](#deployment)
     - [Creating and configuring the EKS cluster](#creating-and-configuring-the-eks-cluster)
     - [Deploying to the cluster](#deploying-to-the-cluster)
     - [Creating the CI/CD pipeline](#creating-the-cicd-pipeline)
   - [Footnotes](#footnotes)


### API specification
The API has just three endpoints:
1. `GET '/'`: This is a simple health check, which returns the response 'Healthy'.
2. `POST '/auth'`: This takes an email and password as JSON arguments and returns a JWT based on a custom secret.
3. `GET '/contents'`: This requires a valid JWT, and returns the decrypted contents of that token.

In other words, you can use `/auth` to get a token and `/contents` to decrypt it. 

The API relies on a secret to handle the decoding. This environment variable is called `JWT_SECRET`. In development, simply `export` it. In production, create a file called `.env_file`.

Additionally, we'll use Gunicorn to make the API production-ready.

## Demo 
In this demo, I have a working EKS cluster with an external IP exposed. This API will work as a microservice for getting JWT tokens. 

The DNS for my cluster is `a7974986dce2b4782bf5c5cf81ee3824-2021521826.us-east-2.elb.amazonaws.com` and the application is running on port 80, so I can do a health check with curl:

![alt text](/readme_assets/health_check.png)

Awesome, now let's get a JWT token using the `/auth` endpoint. Take note, this is my professional work address so please don't tell anyone!

![alt text](/readme_assets/auth.png)

Finally, we can decode the token using our `/contents` endpoint...

![alt text](/readme_assets/contents.png)

## Walkthrough
This walkthrough will cover:

- Containerising the API 
- Testing the endpoints
- Creating an EKS cluster with necessary permissions
- Deploying resources with CloudFormation
- Setting up a CI/CD pipeline from your repo 

### Prerequisites
You'll need

- An AWS account
- AWS CLI configured
- `kubectl` 1.28 and `eksctl`
- Docker 

### Testing  
To deploy the API, we'll first need to containerise it. We'll use Docker. 

Create `.env_file` and set `JWT_SECRET` to a value of your choice.

Next, build the image, e.g. `docker build -t flask_api .`

Optionally, you can test it locally, by running the container:

`docker run --name api_container --env-file=.env_file -p 80:8080 flask_api`. 

To test the three endpoints:

**`/`:** 

```bash
curl localhost:80
```


**`/auth`:**
```bash
export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"WindowsPwd"}' --header "Content-Type: application/json" -X POST localhost:80/auth  | jq -r '.token'`
```

This will save the token into an environment variable. Verify it by running 

```bash
echo $TOKEN
```

**`/contents`**:

(Assuming you have `jq` installed, but not required):

```bash
curl --request GET 'http://localhost:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
```
This should return the same data we passed to `/auth`. 

### Deployment
#### Creating and configuring the EKS cluster
We'll create our EKS cluster with 

```bash
eksctl create cluster --name simple-jwt-api --nodes=2 --version=1.27 --instance-types=t2.medium --region=us-east-2
```

This will create a cluster called `simple-jwt-api` and a node group consisting of two nodes.[1]

Next, get your AWS account ID with 
`aws sts get-caller-identity --query Account --output text`
and add it to `trust.json`. 

Create a new role:
```
aws iam create-role --role-name MyEKSRoleName --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'
```

Attach the policy defined in `iam-role-policy.json` to it:
```
aws iam put-role-policy --role-name MyEKSRoleName --policy-name eks-describe --policy-document file://iam-role-policy.json
```

We'll also need to authorise CodeBuild to use K8s role-based access control (RBAC). Get your cluster's ConfigMap with 

`kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml`

This will save it to a local file. Open it and ensure it looks like the `aws-auth-patch.yml` file in this repo, updating line 13 with the ARN of the role you created.

Apply the updated config with

`kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"`

#### Deploying to the cluster
To enable deployment of the cluster, we want to link the `main` branch of a repo to the CI/CD pipeline. Generate a GitHub access token with all `repo` and `admin:org` scopes. Save it somewhere secure.

#### Creating the CI/CD pipeline
Edit `ci-cd-codepipeline.cfn.yml`, ensuring 

   1. `EksClusterName`
   2. `GitSourceRepo` 
   3. `GitBranch` 
   4. `GitHubUser`
   5. `KubectlRoleName`

All match your particular configuration. Don't do anything with `GitHubToken` yet, we don't want to store this in git.

To create the CloudFormation stack, in the AWS Console

   1. Go to CloudFormation and select Create stack 
   2. Upload the template
   3. Give the stack a name
   4. **Add the access token**
   5. Create the stack 

Now, we'll also save the JWT secret in AWS Parameter store with 
`aws ssm put-parameter --name JWT_SECRET --overwrite --value "myjwtsecret" --type SecureString`. 

Note that the first time, the cluster will need to expose an external IP. You can either run 

`kubectl expose deployment simple-jwt-api --type=LoadBalancer` in your terminal or add it to `buildspec.yml` the first time and then commend it out for subsequent pipeline runs, as I have done.

Now simply make a push to your repository and watch the deployment happen!

Once deployed and the external IP is exposed, you can get a DNS name by running

`kubectl get services simple-jwt-api -o wide`

For a demonstration, see above.

### Footnotes
[1] k8s is overkill for a microservice but useful to use here in case you want to use this as a template for a more complex API or web app.