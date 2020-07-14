# Amazon-EKS

Hii All !!
In this project, I have lunched a crash proof rescalable Ghost Architecture on Amazon EKS. Ghost is a famous open source blogging platform used widely by independent journalists & writers across the globe.

# What is Amazon EKS ?

Amazon Elastic Kubernetes Service (Amazon EKS) is a fully managed Kubernetes service. Customers such as Intel, Snap, Intuit, GoDaddy, and Autodesk trust EKS to run their most sensitive and mission critical applications because of its security, reliability, and scalability.

EKS is the best place to run Kubernetes for several reasons. First, you can choose to run your EKS clusters using AWS Fargate, which is serverless compute for containers. Fargate removes the need to provision and manage servers, lets you specify and pay for resources per application, and improves security through application isolation by design. Second, EKS is deeply integrated with services such as Amazon CloudWatch, Auto Scaling Groups, AWS Identity and Access Management (IAM), and Amazon Virtual Private Cloud (VPC), providing you a seamless experience to monitor, scale, and load-balance your applications. Third, EKS integrates with AWS App Mesh and provides a Kubernetes native experience to consume service mesh features and bring rich observability, traffic controls and security features to applications. Additionally, EKS provides a scalable and highly-available control plane that runs across multiple availability zones to eliminate a single point of failure.
EKS runs upstream Kubernetes and is certified Kubernetes conformant so you can leverage all benefits of open source tooling from the community. You can also easily migrate any standard Kubernetes application to EKS without needing to refactor your code.

# Let's begin -

**Step - 1:** First of all, we need to launch our EKS cluster on AWS Cloud. I have launched a simple cluster with with 3 nodes. The YML code is as follows -

      apiVersion: eksctl.io/v1alpha5
      kind: ClusterConfig

      metadata:
        name: mycluster
        region: us-east-1

      nodeGroups:
         - name: ng1
           desiredCapacity: 3
           instanceType: t2.micro
           ssh: 
                  publicKeyName: sparsh 
           
If you want, you can also launch a Fargte Cluster which will provide you a serverless compute. The YML code to launch a simple Fargate cluster is as follows -

      apiVersion: eksctl.io/v1alpha5
      kind: ClusterConfig

      metadata:
        name: far-spcluster
        region: us-east-1

      fargateProfiles:
        - name: fargate-default
          selectors:
           - namespace: kube-system
           - namespace: default
           
          
  Our EKS cluster has been successfully launched -
  
  ![](/eksproject/eks.png)
  
  ![](/eksproject/eks2.png)
           
  ![](/eksproject/cluster.png)
  
After this, I run the following command in my command prompt -

    aws eks update-kubeconfig --name mycluster
    
This will automatically create a Config file for the mentioned Cluster.

**Step - 2:** Create an EFS storage which can be used by the PVC later. We prefer EFS storage because EBS storage can't be mounted to an instance running in any other subnet. This problem is solved by EFS. I have created an EFS system with the default security group coz I have modified the default security group to allow all incoming & outgoing traffic. **Remember to connect the cluster VPC & correct security group otherwise your deployments will remain pending later on.**

![](/eksproject/efs.png)

![](/eksproject/efs2.png)


**Step - 3:** Next, we create an EFS provisioner with the help of YML code. I have made a separate namespace called _**sparsh**_ & I have done all furthur work in the same namespace. In this code, i have provided the ID & DNS from my already created EFS system. The code to create an EFS provisioner is as follows -

      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: efs-provisioner
      spec:
        selector:
          matchLabels:
            app: efs-provisioner
        replicas: 1
        strategy:
          type: Recreate
        template:
          metadata:
            labels:
              app: efs-provisioner
          spec:
            containers:
              - name: efs-provisioner
                image: quay.io/external_storage/efs-provisioner:v0.1.0
                env:
                  - name: FILE_SYSTEM_ID
                    value: fs-e8ab886b
                  - name: AWS_REGION
                    value: us-east-1
                  - name: PROVISIONER_NAME
                    value: lw-course/aws-efs
                volumeMounts:
                  - name: pv-volume
                    mountPath: /persistentvolumes
            volumes:
              - name: pv-volume
                nfs:
                  server: fs-e8ab886b.efs.us-east-1.amazonaws.com
                  path: /
                  
**We also need to go inside our nodes & run this command once - _yum install amazon-efs-utils_. Go inside the nodes via ssh & runn this command. This will install the necessary softwares on those nodes so that they can be compatible with the EFS Storage.**
                  
**Step - 4:** Next, I have created a YML code to modify some permisssions using ROLE BASED ACCESS CONTROL (RBAC). The code is as follows -

        ---
        apiVersion: rbac.authorization.k8s.io/v1beta1
        kind: ClusterRoleBinding
        metadata:
          name: nfs-provisioner-role-binding
        subjects:
          - kind: ServiceAccount
            name: default
            namespace: sparsh
        roleRef:
          kind: ClusterRole
          name: cluster-admin
          apiGroup: rbac.authorization.k8s.io
          
    
**Step - 5:** Now, I have created a Storage Class which will create a PVC as well as a dynamic PV. I have created two PVCs of 10 Gib each. You can create as per your requirement.

        kind: StorageClass
        apiVersion: storage.k8s.io/v1
        metadata:
          name: aws-efs
        provisioner: lw-course/aws-efs
        ---
        kind: PersistentVolumeClaim
        apiVersion: v1
        metadata:
          name: efs-ghost
          annotations:
            volume.beta.kubernetes.io/storage-class: "aws-efs"
        spec:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 10Gi
        ---
        kind: PersistentVolumeClaim
        apiVersion: v1
        metadata:
          name: efs-mysql
          annotations:
            volume.beta.kubernetes.io/storage-class: "aws-efs"
        spec:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 10Gi
              
              
Here is a screenshot of the process :
 
 ![](/eksproject/pre.png)


**Step - 6:** Next, I have created a secret using YML code, so that I don't have to reveal my passwords & other crucial info later in the Ghost & MYSQL launching codes. Here, I am giving an example of a secret creation but not putting in the actual values. Note that the passwords need to Base 64 Encoded before they are put in a secret.

      apiVersion: v1
      kind: Secret
      metadata:
        name: mysecret
      data:
          dcpassword: cmVkaGF0
          dcdatabase: cHJvamVjdF9kYg==
          sqlrootpassword: cm9vdHBhc3M=
          sqluser: c3BhcnNo
          sqlpassword: cmVkaGF0
          sqldb: cHJvamVjdF9kYg==


**Step - 7:** We need to launch a MYSQL which will be connected to the Ghost Architecture. For this purpose, I have used MYSQL version 5.7. I have picked up the Passwords from the pre-created secret file. The YML code is as follows-


      apiVersion: v1
      kind: Service
      metadata:
        name: ghost-mysql
        labels:
          app: ghost
      spec:
        ports:
          - port: 3306
        selector:
          app: ghost
          tier: mysql
        clusterIP: None
      ---
      apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
      kind: Deployment
      metadata:
        name: ghost-mysql
        labels:
          app: ghost
      spec:
        selector:
          matchLabels:
            app: ghost
            tier: mysql
        strategy:
          type: Recreate
        template:
          metadata:
            labels:
              app: ghost
              tier: mysql
          spec:
            containers:
            - image: mysql:5.7
              name: mysql
              env:
              - name: MYSQL_ROOT_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: mysecret
                    key: sqlrootpassword
              - name: MYSQL_USER
                valueFrom:
                  secretKeyRef:
                    name: mysecret
                    key: sqluser
              - name: MYSQL_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: mysecret
                    key: sqlpassword
              - name: MYSQL_DATABASE
                valueFrom:
                  secretKeyRef:
                    name: mysecret
                    key: sqldb


              ports:
              - containerPort: 3306
                name: mysql
              volumeMounts:
              - name: mysql-persistent-storage
                mountPath: /var/lib/mysql
            volumes:
            - name: mysql-persistent-storage
              persistentVolumeClaim:
                claimName: efs-mysql


**Step - 8:** Now, it's time to launch our Ghost Architecture & connect it to our MYSQL Database. Here also, I have picked up the Passwords & other crucial data from the pre-created secret file that we had created in Step 6. The YML code is as follows-


        apiVersion: v1
        kind: Service
        metadata:
          name: ghost
          labels:
            app: ghost
        spec:
          ports:
            - port: 80
          selector:
            app: ghost
            tier: frontend
          type: LoadBalancer
        ---
        apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
        kind: Deployment
        metadata:
          name: ghost
          labels:
            app: ghost
        spec:
          selector:
            matchLabels:
              app: ghost
              tier: frontend
          strategy:
            type: Recreate
          template:
            metadata:
              labels:
                app: ghost
                tier: frontend
            spec:
              containers:
              - image: ghost:1-alpine
                name: ghost
                env:
                - name: database_client
                  value: mysql
                - name: database_connection_host
                  value: db_ghost
                - name: database_connection_user
                  value: sparsh
                - name: database_connection_password
                  valueFrom:
                    secretKeyRef:
                      name: mysecret
                      key: dcpassword
                - name: database_connection_database
                  valueFrom:
                    secretKeyRef:
                      name: mysecret
                      key: dcdatabase
                ports:
                - containerPort: 2368
                  name: ghost
                volumeMounts:
                - name: ghost-persistent-storage
                  mountPath: /var/lib/ghost
              volumes:
              - name: ghost-persistent-storage
                persistentVolumeClaim:
                  claimName: efs-ghost


Here is a screesnshot showing how these files were launched- 

 ![](/eksproject/done.png)


Now that our Ghost Architecture is launched, we can access it using the DNS & use it -


![](/eksproject/gh.jpg)

Aloha !! We have successfully deployed a rescalable & Crash Proof Ghost Architecture on Amazon EKS.
