# Reference
- [AWS EKS GUIDE](https://aws-eks-web-application.workshop.aws/ko/)

# Step
1. Create cluster (eksctl)
    1) create cluster
       ```
       eksctl create cluster -f clusters/{clusterName}/{clusterName}-cluster.yaml
       ```
    2) check nodes
       ```
       kubectl get nodes
       ```
2. Create IAM OIDC provider
    1) create oidc provider
        ```
        eksctl utils associate-iam-oidc-provider \
        --region ${AWS_REGION} \
        --cluster ${clusterName} \
        --approve
       ```
    2) check oidc provider url
        ```
       aws eks describe-cluster --name ${clusterName} --query "cluster.identity.oidc.issuer" --output text
       ```

3. Create IAM Policy if (withAddonPolicies.albIngress = false)
    1) create iam policy
        ```
       aws iam create-policy \
       --policy-name AWSLoadBalancerControllerIAMPolicy \
       --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
       ```

    2) create service account
        ```
       eksctl create iamserviceaccount \
       --cluster ${clusterName} \
       --namespace kube-system \
       --name aws-load-balancer-controller \
       --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
       --override-existing-serviceaccounts \
       --approve
       ```

4. Deploy Cert Manager
   [Cert manager releases](https://github.com/jetstack/cert-manager/releases)
    ```
    $ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
    ```

5. Deploy AWS Load Balancer Controller
   [ALB Controller releases](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases)
    1) download alb controller yaml
        ```
        # ❗edit --cluster-name
        # ❗delete ServiceAccount yaml spec (kind: ServiceAccount)
        # ❗️edit apiVersion cert-manager.io/v1alpha2 → cert-manager.io/v1
        $ curl https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.4/docs/install/v2_2_1_full.yaml > v2_2_1_full.yaml
        ```
    2) deploy alb controller
        ```
        kubectl apply -f v2_2_1_full.yaml
        ```

    3) check alb controller
        ```
        # check deployment
        kubectl get deployment -n kube-system aws-load-balancer-controller
        # check service account
        kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
        ```

    4) check addon pod's logs
        ```
        kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o "aws-load-balancer[a-zA-Z0-9-]+")
        # detail logs
        ALBPOD=$(kubectl get pod -n kube-system | egrep -o "aws-load-balancer[a-zA-Z0-9-]+")
        kubectl describe pod -n kube-system ${ALBPOD}
        ```

6. Deploy External DNS
   [IAM permissions](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-permissions)
   [Setup External DNS](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/v2.2.4/docs/guide/integrations/external_dns.md)
    1) download external-dns yaml
        ``` 
        # ❗️edit rbac.authorization.k8s.io/v1beta1 → rbac.authorization.k8s.io/v1
        $ curl https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.4/docs/examples/external-dns.yaml > external-dns.yaml
        ```

    2) create iam policy & role and add annotations(role arn) in external-dns.yaml [guide](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-permissions)
       Then apply one of the following manifests file to deploy ExternalDNS. You can check if your cluster has RBAC by `kubectl api-versions | grep rbac.authorization.k8s.io`.

    3) deploy external-dns
        ```
        kubectl apply -f external-dns.yaml
        ```

    4) check deployed successfully
        ```
        kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
        ```

    5) apply the manifest (ingress-controller)
        ```
        annotations:
          kubernetes.io/ingress.class: alb
          alb.ingress.kubernetes.io/scheme: internet-facing
        
          # for creating record-set
          external-dns.alpha.kubernetes.io/hostname: my-app.test-dns.com # give your domain name here
        ```

7. Deploy Deployment, Service, Ingress manifest
    ```
    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml
    kubectl apply -f ingress.yaml
    ```
   
8. Autoscaling Pod & Cluster
   1. HPA
      [Guide](https://aws-eks-web-application.workshop.aws/ko/100-scaling/100-pod-scaling.html)
   2. CA
        `AWS Console > EC2 > Auth Scaling groups > Detail > Edit > Maximum capacity`

## SSL Redirect - AWS Load Balancer Controller
[Redirect Traffic from HTTP to HTTPS](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/tasks/ssl_redirect/)

## Resource quotas
[Docs](https://kubernetes.io/ko/docs/concepts/policy/resource-quotas/)

## Kubernetes dashboard
[Docs](https://github.com/kubernetes/dashboard)
1. Installation
    1) download kubernetes-dashboard yaml
       ```
       kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
       ```

    2) create admin user
       [Guide](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md#creating-a-service-account)

2. Deploy ingress
3. Get token
    ```
    kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
    ```

## Metrics server
1. Installation (latest version auto)
    ```
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```

2. Commands
    ```
    kubectl top pod -n kube-system
    kubectl top node
    ```

## k6
[Docs](https://k6.io/docs/)

1. Examples
    ```
    k6 run script.js
    k6 run --vus 10 --duration 30s script.js
    ```
