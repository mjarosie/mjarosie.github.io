---
categories: ["Dev"]
tags: ["Kubernetes", "AWS", "EKS"]
date:  2021-09-15
layout: post
title:  "IAM roles for Kubernetes service accounts - deep dive"
---

Some time ago I first had anything to do with mixing IAM roles with Kubernetes service accounts. I had to figure out how to retrieve data from DynamoDB from a Pod running in our Kubernetes cluster. Initially I've found it very confusing and as I'm not fond of not understanding what exactly is happening in my code, I've decided to do a bit of research so that I could get my head around it, and document my understanding of it as I'm progressing. In this post I'll show you the nuts and bolts of how IAM and Kubernetes work together in harmony to provide you with a great experience of calling AWS services from your pods with no hussle. Grab a cuppa, as it's gonna be a wild ride through breathtaking steppes of AWS EKS and fascinating plains of Kubernetes.

# Introduction

Consider this scenario: you've written your great service logic in one of your favourite languages, there you've used an AWS SDK for calling some AWS service (say DynamoDB) for retrieving some very important data from there. When you invoke the code locally it works, as you've [configured your AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config) by providing it with **access key ID** and **secret access key** (think of these as username and password). These credentials are associated with an *IAM User* that you've called "Donald Duck" because you're such a great fan you've surrounded yourself with [Donald Duck pocket books](https://en.wikipedia.org/wiki/Donald_Duck_pocket_books) since you were a little kid. The app that you're running locally uses these credentials (and as a result, "*assumes your identity*") when invoking AWS APIs. Those APIs respond with data because your *IAM User* (that your credentials are associated with) have access (hopefully) to AWS resources defined in your AWS account.

{% include image.html url="/assets/2021-09-11-iam-roles-for-kubernetes-service-accounts-deep-dive/01-calling-aws-from-local-2.png" description="The process of calling AWS resources from your local machine." %}

The story is a little bit different when you dockerise the app, publish the image to some docker images repository (say [Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html) - ECS or [DockerHub](https://hub.docker.com/)), and create a Pod in your [EKS](https://aws.amazon.com/eks/) Kubernetes cluster. It's quite obvious that by default the Pod itself doesn't have any AWS-specific credentials associated with it, so it's got no way of calling AWS services without being brutally rejected. When the Pod gets deployed and the app logic arrives to the point where AWS services have to be initialised/called - your application will throw a wobbly (and an exception) and probably exit very ungraciously unless you were far-sighted enough to handle this scenario.

{% include image.html url="/assets/2021-09-11-iam-roles-for-kubernetes-service-accounts-deep-dive/02-calling-aws-from-EKS-using-what.png" description="A Pod calling AWS services authenticating with... what?" %}

It happens because by default the AWS SDK that you depend on to call the AWS service will try to look for credentials in a few different places, such as parameters, environment variables, shared credential profiles file (`~/.aws/credentials`) etc. The exact order and places where credentials are searched for differ between SDKs (to give you two examples, I'll refer to the documentation: [Java](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/credentials.html#credentials-default), [Python](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html)).

# IAM doesn't trust service accounts, do you?

As it happens, every Pod that gets deployed to a Kubernetes cluster (with more or less default configuration) will have a **service account** associated with it. "What's a service account?" you ask, and I'm glad you do! It's a way of giving and "identity" to a process that runs on a specific Pod. In simple words it means that each pod will have its own "username" and "password" (actually it's just a token, but the analogy holds). By default, every Pod in your cluster will be associated with a single service account called... well, "`default`".

Where could it prove useful? As a result, Pod can use these credentials to call cluster's [apiserver](https://kubernetes.io/docs/concepts/overview/components/#kube-apiserver), and the apiserver will know exactly which Pod (or actually - which service account) is calling it. That's quite useful if you want to restrict what Pods can and cannot do when calling the Kubernetes control plane API.

I can already sense your enthusiasm, as you're thinking: "We've got some credentials, why don't we use these to call AWS services as well!". It's not that simple unfortunately. Consider that scenario, imagine we actually called some AWS API with Pod's credentials. How could AWS know where did these credentials come from and whether these can be trusted or not?

{% include image.html url="/assets/2021-09-11-iam-roles-for-kubernetes-service-accounts-deep-dive/03-calling-aws-from-EKS-using-service-account-credentials.png" description="Using service account token to call AWS services. I've used ~ as a token symbol because it looks like a snake, and AWS thinks this token is sneaky - close enough." %}

# Let's jot it down

How do these credentials actually look like? As I've mentioned, by default every Pod will have a service account associated with it. Even though I said that you can think of these credentials as "username" and "password", it's actually an obscure piece of text, called a token. This token will be avialable in the Pod as a file in `/var/run/secrets/kubernetes.io/serviceaccount`. If you spin up and [get a shell access](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/) to some Pod in your cluster (or any cluster - I've followed [this tutorial](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/) using Katakoda) and retrieve a token by running `cat /var/run/secrets/kubernetes.io/serviceaccount/token` it will result in something similar displayed on your screen:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IkxTaWxCV2cwT09uX3JzX2pVbDJWQmZjSVFEajNmbWQ5RERxWllOaDN2ZzAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4teDRzamsiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjQ4OGQzOTVmLTM0YjAtNGFhYi04NzFkLTE3MDg3ZmMwMjdmNyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.s3FwEJaTmnPtW7puzfST4HJCeFefM2qoI1HaBTpd5gQN57YlL5dIyMtUCo6N3NuF-ELHWRd-Z2rMh3YUxD0alQsVqgUWBDFieU7i-hcjLWaGtnLCsxIA4UMCsVkIcBZGAEDYPLSZ2KANLPFktGvqEdAqr1CD9MGyu-dSvCWMOLGs-1RRTykRWS9nC3ntrF1kk400hqjO5EAkWk8Bk63Y6ZAgCrGzQmlu71tGkFGPSeBN_eDwVKYhEmQMQ2CLxUFDuOHLXNX5iinL-B5qBObtHECyn2WvogjNalQiOZIg93cARrB5fgv2dmJb2wYYfz01xvK7RPvX21Il4nXXGRO6pA
```

Doesn't look pretty, huh? As it turns out, this piece of text adheres to what's called a [JWT](https://datatracker.ietf.org/doc/html/rfc7519) (which stands for "JSON Web Token" and is usually pronounced "jot"). Because it's not *encrypted*, but just *encoded,* we can decode it and actually try to understand its content. If you go to [https://jwt.io/](https://jwt.io/) and paste the token in there, we'll see that the "payload" part of the token looks something like this:

```json
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "default",
  "kubernetes.io/serviceaccount/secret.name": "default-token-x4sjk",
  "kubernetes.io/serviceaccount/service-account.name": "default",
  "kubernetes.io/serviceaccount/service-account.uid": "488d395f-34b0-4aab-871d-17087fc027f7",
  "sub": "system:serviceaccount:default:default"
}
```

What you can see above is what's usually called the **claims** of the token. From what the token "claims" we can conclude that:

- The issuer of the token is `kubernetes/serviceaccount` (that's who has **issued**, or created the token)
- The subject of the token is `system:serviceaccount:default:default` (that's who the token was issued **to**)
- It's got some other kubernetes-specific claims in it

# Issues on top of issues

Looking at the token above we've encountered the key word: *issued*. What is this all faff about? Let's start with an issuer. It's a trusted entity in the system which is responsible for *issuing* tokens. Issuing tokens means creating (you can also say "minting") these tokens and handing them out to interested parties (such as our intrepid Pod) that are allowed to get it in the first place. Think of all these tokens that you buy when you go to a funfair. Having them allows you to access different attractions such as hoping on that carousel or getting into the haunted house (and never coming back...). But in order to be allowed to get these tokens, you need to do something first - PAY!

In the funfair example - the issuer would be the funfair organiser, or perhaps the company that the funfair organiser paid to handle all this tokens nonsense (it technical terms this process is called "federation", we'll get to that soon). In the JWT token example above the token issuer was some entity which calls itself `kubernetes/serviceaccount`. The issue with this issuer (hehe) and tokens issued by it is that AWS doesn't put any trust in it (because why should it?).

Another quite important problem with the service account token is that it doesn't expire - if you've seen some JWT tokens in your lifetime, you might have noticed that the one above is missing an `exp` claim, which usually indicates the expiration time of the given token. Long-lived token is a big no-no in the access management world, as it poses some security risk in case the token is lost/stolen. Have you ever lost your keys? If you did - I bet you didn't feel secure until the locks were changed, and the process of changing locks itself is quite a faff. Imagine losing keys which can open the doors to your house forever - even after changing locks and getting a new pair of keys. Tragedy!

Looking at all these difficulties, you might be wondering why am I talking about all these services and tokens if it seems like that'd be useless for achieving our goal: calling AWS services from Pods. Stay with me, we're getting there!

# Federated identities

In 2014, AWS Identity and Access Management (IAM) added support for *federated identities* using OpenID Connect (OIDC). What does it mean? Your AWS account, apart from trusting credentials that it issues by itself, can be set to trust some other entity (called **identity provider**) - to **federate** (remember the funfair tokens example?) the process of authenticating and issuing tokens. Whoever authenticates with this identity provider could then try to access some AWS **resources**. In our case the **subject** is our Pod and **resources** are either DynamoDB or S3 bucket. To make it a little bit less abstract - I'm going to forget about the Kubernetes cluster for a moment and use an example of a mobile game.

Imagine you're building a game of Snake for mobile devices. One of its features is storing some player and score information in S3 and DynamoDB. You're a lazy developer so obviously you don't want to be bogged down with implementing the whole custom sign-in code or managing users' identities. Guess what - you're lucky! You can just ask your users to sign in using one of the well-known external identity providers (IdP), such as Google, Facebook, Twitter, Amazon etc. There's no need for managing usernames, passwords, emails and all that jazz - the big corp is doing it for you!

Now image one of the first users of your application - Alice, logs in with an identity provider of her choice - Google. When that login succeeded (she had to provide valid username and password) - Google can just issue a token that now belongs to Alice and provide it to your mobile app (it might be a bit more complicated than that - but let's not get into too much details - if you want to know more read about [Authorization Code Flow with Proof Key for Code Exchange (PKCE)](https://auth0.com/docs/authorization/flows/authorization-code-flow-with-proof-key-for-code-exchange-pkce)).

{% include image.html url="/assets/2021-09-11-iam-roles-for-kubernetes-service-accounts-deep-dive/04-alice-logging-in-with-google.png" description="Alice logging in with Google identity provider." %}

That's not it yet! From AWS' perspective, the token provided by Google is called `WebIdentityToken`. There's still one step that needs to be done to complete our journey and it's the process of swapping this token for temporary AWS security credentials.

# Swap That Swiftly

When someone (or something) accesses AWS services such as DynamoDB or S3 bucket, usually AWS requires the caller to provide a set of credentials (**access key ID** and **secret access key**) to establish whether a resource can be accessed or not. You've already seen it in practice when we discussed the example of calling DynamoDB and S3 from your local machine - the app was smart enough to use your AWS credentials (again: **access key ID** and **secret access key** associated with your "Donald Duck" IAM User) for doing it. Similar credentials need to be used when your Snake game/Pod deployed to your cluster needs to access some interesting nuggets stored in a DynamoDB table or an S3 bucket.

So far we've ended up with a token issued to Alice by her favourite identity provider. What makes it possible to exchange it for a set of short-lived AWS credentials that can be used for calling AWS web services is an AWS service called *Security Token Service* (**STS** for short). We're going to invoke STS API operation called [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html) to complete this exchange.

Take a look at the diagram below. When developing the application, we've created a special IAM role which allows accessing DynamoDB and/or some S3 bucket. When creating this Role we've instructed it to trust Google as a federated identity provider (step "0"). Alice logs in with Google, which provides her (well, actually the application running on her mobile phone) with an access token `(*)` which we call Web Identity token. Then the app will exchange this Web Identity token for temporary security credentials `(#)` that are associated with the IAM role that was just mentioned above (the application has to provide the ARN of the assumed IAM role as a parameter when invoking `AssumeRoleWithWebIdentity` STS API operation). Finally, these credentials can be used to actually call AWS web services such as DynamoDB or S3.

{% include image.html url="/assets/2021-09-11-iam-roles-for-kubernetes-service-accounts-deep-dive/05-alice-logging-in-with-google-full-journey-2.png" description="The process of Alice using Snake app, which in turn calls DynamoDB and S3." %}

You might be wondering: "How the hell did we end up talking about Snake game on a mobile phone, I'm elbows deep in Kubernetes config, get to the point!". But we're almost there!

As now you've got an understanding of how identity federation works in AWS, we can replace the bits we talked about above with Kubernetes-related concepts:

- Alice becomes a service account (and provides an app with credentials that the identity provider will accept)
- Snake app becomes your Pod (and exchanges credentials for token and token for temporary AWS credentials)
- Google becomes a cluster-specific identity provider that Kubernetes enables you to spin up

Here's the result, compare it with the previous diagram and you'll notice that it's almost the same:

{% include image.html url="/assets/2021-09-11-iam-roles-for-kubernetes-service-accounts-deep-dive/06-eks-iam-roles-for-service-account-2.png" description="The process of Pod authenticating with service account credentials and calling DynamoDB and S3." %}

## Making this work in your cluster

After this lengthy introduction what's left now is to go through the details of how to set this up in your cluster and what's actually causing this all to work.

### OIDC Identity Provider setup

From your side you're required to set up an OIDC identity provider for your EKS cluster, but it's quite straightforward. Actually - AWS has created an OIDC in your EKS cluster for you already! You can either retrieve it from the AWS Console (in cluster's configuration page, it's a field called "OpenID Connect provider URL") or if you prefer the command line, you can invoke: `aws eks describe-cluster --name $CLUSTER_NAME --query cluster.identity.oidc` (obviously, you need to define `CLUSTER_NAME` environment variable first). Then all you need to do is to create an Identity Provider in IAM settings so that it matches the URL of the identity provider that's there in your cluster. For details, refer to the [documentation](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html). If you've got your cluster Terraformed, refer to [this article](https://marcincuber.medium.com/amazon-eks-with-oidc-provider-iam-roles-for-kubernetes-services-accounts-59015d15cb0c) on how to do it correctly.

### IAM role setup

The next step is to make sure that the IAM role that you'll define (so that your Pods can assume it) actually trusts the cluster's OIDC identity provider.

A quick reminder on IAM roles - any meaningful IAM role is defined by at least two policies. One is the **trust policy** that specifies who can assume the role. All other policies are **permissions policies** that specify the AWS actions and resources that whoever assumes this role is allowed or denied access to.

**Trust policy**

An example trust policy that you provide when you're creating your IAM role in this scenario looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Principal": {
        "Federated": "<ARN of your cluster's OIDC Identity Provider>"
      },
      "Condition": {
        "StringEquals": {
          "<URI of your cluster's OIDC Identity Provider>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Pay attention - there are two different identifiers that you need to provide here!

`Statement.Principal.Federated` - that's the **ARN** of the Identity Provider that you've just defined in IAM (just above, do you remember?)

`Statement.Condition.StringEquals` - the key should contain the **URI** originally given by the EKS cluster (you've also used it to create aforementioned Identity Provider in IAM)

**Permission policy**

An example permission policy could look like this (obviously, that'll be application specific, here I'm just giving an example):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "dynamodb:Query",
                "dynamodb:ListTables",
                "dynamodb:GetItem",
                "dynamodb:BatchGetItem"
            ],
            "Resource": [
                "arn:aws:dynamodb:<AWS region>:<your AWS account ID>:table/<your table name>"
            ]
        }
    ]
}
```

### Off the hook

The only mistery left is: what's putting it all in motion? You don't want to be putting any logic in your Pod application that does any crazy token exchanges or anything like that! Meet [Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook/).

To do its job properly, the webhook (installed and running in your EKS cluster by default) will be looking for a very specific thing: a `ServiceAccount` Kubernetes resource annotated with `eks.amazonaws.com/role-arn` annotation. When it finds such a `ServiceAccount`, all new Pods launched using this `ServiceAccount` will be modified to use IAM for Pods. What does it mean in practice? When it starts, your Pod will automatically get:

- some environment variables (`AWS_DEFAULT_REGION`, `AWS_REGION`, `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`, `AWS_STS_REGIONAL_ENDPOINTS`)
- a volume (containing a Web Identity Token) mounted to the container

The volume will be mounted under `/var/run/secrets/eks.amazonaws.com/serviceaccount/`. Sounds similar, doesn't it? Some differences between the Kubernetes token (the one we initially discussed) and the AWS Web Identity Token are that:

- Web Identity Token is mounted under a different path (as a reminder, in case of Kubernetes token it was `/var/run/secrets/kubernetes.io/serviceaccount/`)
- Web Identity Token has got an expiration date

So to progress, you'll have to create a Kubernetes `ServiceAccount` resource annotated with the ARN of the IAM role we've created just moments ago. Here's an example configuration if you need an inspiration (obviously, replace the AWS Account ID and the IAM role name):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-serviceaccount
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::111122223333:role/allow-read"
```

Then you'll need to modify the configuration of your Pod and specify a `serviceAccountName` parameter. As per [the documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-multiple-service-accounts):

> To use a non-default service account, set the `spec.serviceAccountName` field of a pod to the name of the service account you wish to use.

So go and do just that!

Next time your pod gets recreated, you'll be able to see the fruits of your work! Just ssh again into the Pod that's using our brand new, fresh and shiny `ServiceAccount`, and invoke `cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token`. You can again copy the content of the token and paste it in [https://jwt.io/](https://jwt.io/) to see that its payload looks a bit different:

```json
{
  "aud": [
    "sts.amazonaws.com"
  ],
  "exp": 1626519664,
  "iat": 1626433264,
  "iss": "<your cluster's OIDC URI>",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "<name of your Pod>",
      "uid": "<ID of your Pod>"
    },
    "serviceaccount": {
      "name": "my-serviceaccount",
      "uid": "<ID of your ServiceAccount>"
    }
  },
  "nbf": 1626433264,
  "sub": "system:serviceaccount:default:my-serviceaccount"
}
```

Congratulations, that's something AWS services can finally accept when your application calls their APIs! The AWS SDK will fetch the token from this location when instructed to call AWS services, regularly exchange it for fresh IAM role credentials when they get expired (with the help of STS service) and retrieve required data from DynamoDB and S3. From now everything will go swimmingly. Trust me.

## Summing up

Let's take a step back and think what we went through to make it possible for your Pods to call AWS Services such as DynamoDB or S3.

First we obviously needed an application ready to be running in your cluster.

Then we've created an IAM Identity Provider that points to your cluster's Identity Provider.

Then we've created an IAM role that trusts your cluster's Identity Provider (through the IAM Identity Provider resource mentioned above).

Then we've created a `ServiceAccount` Kubernetes resource.

Then we've modified the specification of the Pod so that is uses this `ServiceAccount`.

Finally (thanks to Amazon EKS Pod Identity Webhook) the Pod automagically gets access to IAM role credentials that it can use to call AWS services!

That's it, hope you enjoyed it!