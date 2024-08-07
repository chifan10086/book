# PX包网EKS环境说明

1. K8S     V1.28
2. Rancher V2.8.5
3. Redis   V7.1.0 (AWS ElastiCache云原生，开2个，1个业务用，1个Plume日志用)
4. Mysql   V5.7   (AWS RDS 云原生，Nacos和XXL-Job用)
5. RabbitMQ V3.11.28 (AWS 云原生)
6. Elasticsearch  V7.1   (Amazon OpenSearch Service 这个用于风控)
7. OpenSearch     V2.13  (Amazon OpenSearch Service 这个用于链路日志系统)
8. Nacos          V2.2.3  (微服务注册配置中心)
9. plumelog       V3.5.3  (看日志的)
10. Zipkin        V3.4     (链路追踪)
11. EFS           用于EKS挂载
12. TIDB

---

### 2、配置AKSK和区域

配置进入AWS控制台，创建IAM用户，附加`AdministratorAccess`的IAM Policy，最后给这个用户生成AKSK密钥。

在安装好AWSCLI的客户端上，进入命令行窗口，执行`aws configure`，然后填写正确的AKSK。同时，在命令的最后一步配置region的时候，设置region为本次测试环境的`ap-southeast-1`。


## 3、安装EKS客户端和Kubectl客户端（三个OS类型根据实验者选择其一）

请注意，eksctl版本和创建EKS的版本有对应关系，因此请升级的客户端的eksctl到最新版本。



### 4、Linux下安装eksctl和kubectl工具

在Linux下安装eks工具，包括eksctl和kubectl两个。

使用X86_64架构的执行如下命令：

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /bin
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl
chmod 755 kubectl
sudo mv kubectl /bin
eksctl version
```

安装完毕后即可看到eksctl版本，同时kubectl也下载完毕。

客户端准备完毕。

## 创建EKS集群的配置文件

EKS集群分成EC2模式和无EC2的Fargate模式。本文为有EC2模式的配置，有关Fargate配置将在后续实验中讲解。在接下来的网络模式又有两种：

### 5、使用现有VPC的子网

#### （1）给EKS要使用的Subnet子网打标签

请确保本子网已经设置了正确的路由表，且VPC内包含NAT Gateway可以提供外网访问能力。然后接下来为其打标签。

注意： AWS为了高可用跨区服务 请至少创建2个public 子网和2个private子网

找到当前的VPC，找到有EIP和NAT Gateway的Public Subnet，为其添加标签：

- 标签名称：`kubernetes.io/role/elb`，值：`1`

接下来进入Private subnet，为其添加标签：

- 标签名称：`kubernetes.io/role/internal-elb`，值：`1`

接下来请重复以上工作，三个AZ的子网都实施相同的配置，注意第一项标签值都是1。

请不要跳过以上步骤，否则后续使用ELB会遇到错误。

#### （2）要求EKS Nodegroup使用特定的Subnet

+ 去AWS的EKS控制面板创建节点组，节点组至少2组或者更多，视项目规划而定
    +  1组选定公有子网
    +  1组选定私有子网

创建完成。

## 6、查看创建结果

aws eks update-kubeconfig --region 区域代码 --name 集群名字


此过程需要10-15分钟才可以创建完毕。执行如下命令查询节点。

```
kubectl get node
```

返回节点表示正常。

控制台创建参照
[https://imroc.cc/kubernetes/basics/deploy/aws/eks]()



## 7.安装Rancer可视化界面管理EKS

```
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  --privileged \
  rancher/rancher:latest
```



### 创建IAM Policy

由于EKS版本的不断升级，AWS Load Balancer Controller的版本也在升级，需要的IAM Policy在一段时间内是稳定的，但时间长了（例如几年过去）也会变化，因此建议用最新的Policy。如果之前在某一个Region使用过EKS服务，那么系统内可能已经有一个旧的Policy了，建议删除替换为新的。

对于IAM Policy，还区分AWS海外区和中国区。请选择要部署的区域，下载对应的IAM Policy文件。

#### (1) 全球区域（海外区域）

从Github下载最新的IAM Policy。

```shell
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```


#### (2) 创建IAM Policy

创建IAM Policy，执行如下命令。如果之前创建过旧版本的策略，那么可以从AWS控制台的IAM中找到这个Policy并删掉它，然后重新创建。注意上一步下载的文件如果是中国区的版本，注意文件名的区别。

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

返回如下结果表示成功。

```
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPAR57Y4KKLBSWJ4A2NG",
        "Arn": "arn:aws:iam::133129065110:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2024-07-03T04:15:12+00:00",
        "UpdateDate": "2024-07-03T04:15:12+00:00"
    }
}
```

此时需要记住这个新建Policy的ARN ID，下一步创建角色时候将会使用

### 3、创建EKS Service Account

执行如下命令。请替换cluster名称、attach-policy-arn、region等参数为本次实验环境的参数。其他参数保持不变。

```
eksctl create iamserviceaccount \
  --cluster=eksworkshop \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::133129065110:policy/AWSLoadBalancerControllerIAMPolicy \ #这里记得改成自己的
  --approve
```

### 4、通过Helm安装Load Balancer Controller

在之前EKS版本上，主要通过`Kubernetes manifest`来安装Load Balancer Controller。为了简化部署过程，后续都推家使用Helm来安装。以前manifest安装方法可从本文末尾的参考文档中获取。

#### （1）操作环境安装Helm


在Amazon Linux 2023下：

```
sudo dnf install helm
```

#### （2）安装Helm的软件库

执行如下命令：

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

#### （3）运行Load Balancer Controller

请替换如下命令的集群名称为真实名称：

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eksworkshop \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```



部署成功返回信息如下：

```
NAME: aws-load-balancer-controller
LAST DEPLOYED: Wed Jul  3 12:21:16 2024
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!
```

#### （4）检查部署结果

在创建AWS Load Balancer Controller后，等待几分钟启动完成，执行如下命令检查部署结果：

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

返回结果如下：

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           88s
```

表示部署成功，控制器已经正常启动。

此时只是配置好了AWS Load Balancer Ingress Controller所需要的Controller对应的Pod，如果现在去查看EC2控制台的ELB界面是看不到有负载均衡被提前创建出来的。每个应用可以在自己的yaml中定义负载均衡器相关参数，然后随着应用的pod一起一起创建负载均衡。


## 三、参考文档

Install the AWS Load Balancer Controller using Helm

[https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html]()

Install the AWS Load Balancer Controller add-on using Kubernetes Manifests

[https://docs.aws.amazon.com/eks/latest/userguide/lbc-manifest.html]()