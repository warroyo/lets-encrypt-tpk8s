# Add let's encrypt support to multicloud ingress trait

These steps are now simplified becuase let's encrypt support is generally available. 


## Using aws credentials

1. update the ingress capabiltiy on the cluster group to use let's encrypt. In the advanced yaml section for the package add the following and apply the change

```yaml
clusterIngressCa:
  acme:
    email: your-email
    server: "https://acme-v02.api.letsencrypt.org/directory"
    solver:
      route53:
        accessKeyID: "aws-accesskey"
        accessKeySecret: "aws-secretkey"
        hostedZoneID: "zone-id"
        region: "us-west-2"
```

2. update the multicloud-ingress trait in the networking profile and set `UseClusterIssuer` to true


## Deploy your app

Now when you deploy an app and httproute you will see that the issuer used is backed by LE and it should creates some dns challenges and eventually generate the cert.  



## Using IRSA in EKS.


1. enable oidc on the eks cluster

```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

2. create the role and policies for certman. run the below step for each cluster you want to enable. it will append the additonal clusters to the trust document

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION=us-west-2
export CLUSTER_NAME=<your cluster name>
export OIDCPROVIDER=$(aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION --output json | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///')

cat dns-trust.json| jq '.Statement  += [{"Effect":"Allow","Principal":{"Federated":"arn:aws:iam::'${AWS_ACCOUNT_ID}':oidc-provider/'${OIDCPROVIDER}'"},"Action":"sts:AssumeRoleWithWebIdentity","Condition":{"StringEquals":{"'${OIDCPROVIDER}':sub":"system:serviceaccount:cert-manager:cert-manager","'${OIDCPROVIDER}':aud":"sts.amazonaws.com"}}}]' > dns-trust.tmp && mv dns-trust.tmp dns-trust.json
```


3. create the role with the trust document in aws.

```bash
aws iam create-role --role-name tpk8s-certman --assume-role-policy-document file://dns-trust.json

```

4. create the permissions policy and attach to the role

```bash

aws iam put-role-policy --role-name tpk8s-certman --policy-name tpk8s-dns --policy-document file://dns-policy.json

```


5. get the role ARN and add it to the cert-man-overlay.yml

```bash
aws iam get-role --role-name tpk8s-certman --query Role.Arn --output text
```


6. apply the overlay secret to the cluster group context. in the `cert-man-overlay.yml` update it with the arn created in the previous step if you havent already.
```bash
export KUBECONFIG=~/.config/tanzu/kube/config  
tanzu project use <your-project>
tanzu ops clustergroup use <your-cluster-group>
k apply -f cert-man-overlay.yml
```

7. update the capability package to use the overlay.

```bash
k apply -f certman-pkg.yml
```

8. update the ingress capabiltiy on the cluster group to use let's encrypt. In the advanced yaml section for the package add the following and apply the change

```yaml
clusterIngressCa:
  acme:
    email: your-email
    server: "https://acme-v02.api.letsencrypt.org/directory"
    solver:
      route53:
        hostedZoneID: "zone-id"
        region: "us-west-2"
```

9. update the multicloud-ingress trait in the networking profile and set `UseClusterIssuer` to true

  