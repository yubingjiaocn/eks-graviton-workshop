# 在EKS上构建混合架构的应用

## 实验简述

在本次实验中，我们会进行以下步骤：
* 利用Porting Advisor for Graviton对现有应用进行扫描
* 利用`docker buildx`构建多架构容器镜像，并上传至ECR
* 在EKS集群中增加基于Graviton的工作节点，并部署应用

## 环境准备

本次我们会提供一个测试账号，该账号下包含一个Cloud9开发环境，预装：

* Docker
* AWS CLI
* git
* kubectl
* eksctl

同时该账号有足够的权限管理EKS集群和工作负载。

访问此环境的步骤如下：

* 在浏览器中打开 https://catalog.workshops.aws/ ，选择**Getting Started**
* 认证方式选择**Email one-time password (OTP)**
* 输入您的邮箱，系统会将一次性密码发送到您的邮箱中
* 输入邮件中的9位密码，完成认证
* 在**Event Access Code**中，输入现场提供的12位Access Code，进入Workshop
* 在左下角的**AWS account access**中，选择**Open AWS console (us-west-2)**，进入AWS控制台。
* 在上方搜索框中，输入**Cloud9**
* 选择**eks-workshop**一行的**Open**，打开Cloud9 IDE
* 运行下列命令以配置环境变量：

```bash
export ECR_URL=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
aws configure set default.region ${AWS_DEFAULT_REGION}
aws configure set default.account ${AWS_ACCOUNT_ID}
aws configure get default.region
aws configure get default.account
```

## 利用Porting Advisor for Graviton对现有应用进行扫描

### 获取样例应用

我们使用[MyBatis JPetStore](https://github.com/mybatis/jpetstore-6)作为示例应用，该应用是由MyBatis 3, Spring 5 和 Stripes构建的Java应用。运行下列命令以获取代码：
```bash
git clone https://github.com/mybatis/jpetstore-6.git
```

### 进行代码扫描

Porting Advisor for Graviton 是一款开源命令行工具，可以进行代码分析并标明对应代码是否可在ARM上构建。我们使用该工具对刚刚下载的源代码进行扫描：

```bash
docker run -it -v $(pwd)/jpetstore-6:/mnt/ public.ecr.aws/bingjiao/porting-advisor-for-graviton:main /mnt
```

可以看到以下输出：

```bash
Porting Advisor for Graviton v1.0.2
Report date: 2023-05-23 13:53:17

43 files scanned.
detected java code. we recommend using Corretto. see https://aws.amazon.com/corretto/ for more details.
detected java code. min version 8 is required. version 11 or above is recommended. see https://github.com/aws/aws-graviton-getting-started/blob/main/java.md for more details.

Report generated successfully. Hint: you can use --output FILENAME.html to generate an HTML report.
```

可以看到，在报告中未出现风险点，该应用可以在Graviton实例上构建和运行。

## 利用Docker buildx构建多架构镜像

通常我们使用 `docker build` 在本机构建镜像。`buildx`是该功能的扩展，能够使用由 [Moby BuildKit](https://github.com/moby/buildkit) 提供的构建镜像额外特性，它能够创建多个 builder 实例，在多个节点并行地执行构建任务，以及跨平台构建。`docker buildx`已在目前安装的Docker版本中提供。

### 配置ARM64架构构建环境

在本次实验中，为方便起见，我们会使用`qemu`模拟器在x86实例上模拟ARM64架构的指令集，从而完成构建。但`qemu`性能较原生构建有损失。在生产环境中不建议使用。

运行以下命令以构建ARM64环境：

```bash
docker run --privileged --rm tonistiigi/binfmt --install arm64
```

运行完成后，可以看到如下输出，证明qemu成功配置：

```bash
installing: arm64 OK
{
  "supported": [
    "linux/amd64",
    "linux/arm64",
    "linux/386"
  ],
  "emulators": [
    "qemu-aarch64"
  ]
}
```

运行以下命令以创建新的Builder:

```bash
docker buildx create --name mybuild --use
docker buildx inspect --bootstrap
```

在输出中可以看到:

```bash
Nodes:
Name:      mybuild0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Buildkit:  v0.11.6
Platforms: linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/amd64/v4, linux/arm64, linux/386
```

证明该Builder可以构建ARM64架构的镜像。

### 构建多架构镜像

运行以下命令创建ECR镜像存储库并登陆：

```bash
aws ecr create-repository --repository-name=jpetstore --region=$AWS_DEFAULT_REGION
aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_URL}
```

利用`docker buildx`构建镜像并上传至ECR：

```bash
cd jpetstore-6
docker buildx build --platform linux/amd64,linux/arm64 -t $ECR_URL/jpetstore:latest --push .
```

构建完成后，我们可以通过`docker buildx imagetools`检查该镜像是否包含多架构：

```bash
docker buildx imagetools inspect $ECR_URL/jpetstore:latest
```

## 在EKS集群中增加基于Graviton的工作节点并部署应用

### 增加Graviton的工作节点

我们可以使用eksctl创建使用Graviton处理器的实例作为工作节点。运行下列命令以生成节点组配置文件：

```bash
cat << EOF > nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${EKS_CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
vpc:
  id: ${VPC_ID}
  securityGroup: ${EKS_CLUSTER_SECURITY_GROUP_ID}
  subnets:
    private:
      ${AZ1}: { id: ${VPC_PRIVATE_SUBNET_ID_0} }
      ${AZ2}: { id: ${VPC_PRIVATE_SUBNET_ID_1} }
      ${AZ3}: { id: ${VPC_PRIVATE_SUBNET_ID_2} }

managedNodeGroups:
  - name: managed-graviton
    instanceType: m6g.xlarge
    desiredCapacity: 1
    privateNetworking: true
EOF
```

配置文件完成生成后，即可创建节点组：

```bash
eksctl create nodegroup -f nodegroup.yaml --skip-outdated-addons-check=true
```

在节点组创建完成后，运行以下命令以列出新创建的节点及其架构：

```bash
kubectl get nodes --label-columns=kubernetes.io/arch

NAME                                         STATUS   ROLES    AGE     VERSION                ARCH
ip-10-42-10-157.us-west-2.compute.internal   Ready    <none>   2m21s   v1.23.17-eks-0a21954   arm64
ip-10-42-10-52.us-west-2.compute.internal    Ready    <none>   8h      v1.23.15-eks-49d8fe8   amd64
ip-10-42-10-73.us-west-2.compute.internal    Ready    <none>   8h      v1.23.15-eks-49d8fe8   amd64
ip-10-42-11-51.us-west-2.compute.internal    Ready    <none>   8h      v1.23.15-eks-49d8fe8   amd64
```

可以看到新创建的arm64架构节点。

### 部署应用，并将其调度到Graviton工作节点

运行以下命令以生成Kubernetes 资源定义：

```bash
cat << EOF > jpetstore.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jpetstore
  labels:
    app: jpetstore
spec:
  selector:
    matchLabels:
      app: jpetstore
  replicas: 1
  template:
    metadata:
      labels:
        app: jpetstore
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
      containers:
        - name: jpetstore
          image: ${ECR_URL}/jpetstore:latest
          ports:
            - name: web
              containerPort: 8080
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
  creationTimestamp: null
  labels:
    app: jpetstore
  name: jpetstore
spec:
  ports:
  - name: web
    port: 80
    targetPort: 8080
  selector:
    app: jpetstore
  type: LoadBalancer
  loadBalancerClass: service.k8s.aws/nlb
EOF
```

部署该应用并检查运行状况：
```bash
kubectl apply -f jpetstore.yaml
kubectl get pod -o wide
```

可以看到，该应用在新创建的Graviton实例上运行。

```bash
NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE                                         NOMINATED NODE   READINESS GATES
jpetstore-7d4db7d744-jlwhd   1/1     Running   0          5m14s   10.42.10.114   ip-10-42-10-157.us-west-2.compute.internal   <none>           <none>
```

可以通过创建的负载均衡器访问该容器：

```bash
kubectl get svc
curl http://<LoadBalancer地址>/jpetstore/
```

为测试多架构镜像的混合调度能力，我们可以将应用调度到x86-64架构上。在Cloud9中，利用内置编辑器修改刚刚创建的jpetstore.yaml，将`arm64`改为`amd64`。修改完成后，运行下列命令以应用：

```bash
kubectl apply -f jpetstore.yaml
kubectl get pod -o wide
```

可以看到，新的Pod在x86_64节点上被创建，应用继续正常运行。
