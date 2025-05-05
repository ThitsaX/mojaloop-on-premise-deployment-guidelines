# EXPERIMENTAL: Mojaloop Control Center On-Premise Deployment Guidelines
Mojaloop deployments are predominantly cloud-based due to their scalability, cost-effectiveness, and inherent security. However, there are instances where organizations opt for a self-hosted on-premise solution due to budget constraints or the readiness of alternative solutions. On-premise setups offer several benefits, including enhanced transparency regarding system limitations and costs, clear ownership of issues, improved visibility into system performance, reduced latency, and the necessity for compliance with regulations that require data to be stored within the country's own data center.

This document provides an overview of the Mojaloop Control Center, focusing on its deployment, particularly in on-premise environments. Understanding the deployment process is essential for organizations looking to leverage Mojaloop for enhancing financial inclusion through interoperable digital financial services.
## Table of Contents
1. [Introduction](#introduction)
2. [Key Technical Requirements](#key-technical-requirements)
3. [Prerequisites](#prerequisites)
4. [Getting Started](#getting-started)
5. [Configuration](#configuration)
6. [Deployment](#deployment)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

## Introduction
The Mojaloop Control Center is a core component of the Mojaloop platform, enabling the management of interoperable digital financial services. This guide outlines the steps to deploy the Control Center on-premise, covering infrastructure setup, configuration, and deployment processes to support financial inclusion initiatives.

---

## Key Technical Requirements

1. **Infrastructure:**

   * Understanding of cloud computing principles, especially regarding scalability, cost-effectiveness, and security.
   * Familiarity with both cloud-based and on-premise deployment models, including their respective advantages and limitations.

2. **Terraform and Terragrunt:**

   * Proficiency in using Terraform for provisioning and managing infrastructure.
   * Knowledge of writing modular and reusable Terraform configurations to facilitate efficient resource deployment.
   * Understanding what Terragrunt is: a thin wrapper for Terraform that provides extra tools for keeping your configurations DRY (Don't Repeat Yourself).
   * Familiarity with how Terragrunt helps manage Terraform configurations across multiple environments and modules.

3. **Ansible:**

   * Experience with Ansible for configuration management and automation of deployment tasks.
   * Ability to create playbooks and manage inventory files for orchestrating software installations and updates.

4. **Containerization and Orchestration:**

   * Understanding of Docker for containerization and Kubernetes (K8s) for orchestration.
   * Familiarity with deploying applications in a microservices architecture using K8s.

5. **Version Control Systems:**

   * Proficient use of Git for version control, including branching, merging, and managing repositories.

6. **Scripting Languages:**

   * Basic knowledge of shell scripting (e.g., Bash) to automate tasks within the deployment process.

7. **Installation and Configuration:**

   * Ability to install necessary tools such as Terraform, Ansible, Git, jq, and yq.
   * Experience in configuring environments through YAML files and other configuration management tools.

8. **Troubleshooting:**

   * Skills in diagnosing issues during deployment processes and applying best practices for troubleshooting common problems.

9. **Networking:**

   * Understanding of networking concepts relevant to deployments, including security groups or firewalls, load balancers, NAT gateways, and domain management.

10. **Compliance and Security:**

    * Awareness of regulatory requirements regarding data storage and processing within specific jurisdictions.
    * Knowledge of security best practices for managing sensitive data.

11. **Monitoring and Observability:**

    * Familiarity with tools for monitoring application performance (e.g., Grafana) and logging (e.g., Loki).
    * Ability to implement observability solutions that provide insights into system performance.

12. **Zitadel (Identity & Access Management):**

    * Understanding of Zitadel, an open-source identity and access management (IAM) solution.
    * Experience in integrating Zitadel for authentication, authorization, and Single Sign-On (SSO) solutions in a cloud-native environment.

13. **NetBird (Zero Trust Networking):**

    * Familiarity with NetBird, a Zero Trust Network Access (ZTNA) solution for secure, seamless networking across distributed environments.
    * Knowledge of how to integrate NetBird for secure communication between resources and services in a multi-cloud or hybrid cloud setup.

---

This updated version now includes **Zitadel** for IAM (Identity and Access Management) and **NetBird** for secure networking as part of the key technical requirements.


## Prerequisites
Before you begin, ensure you have the following installed:
- [terraform](https://www.terraform.io/downloads.html) (v1.3.2)
- [terragrunt](https://github.com/gruntwork-io/terragrunt/releases) (v0.57.0)
- [python3](https://www.python.org/downloads/) (v3.12)
- [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) (ansible-core v2.18.1 or ansible [v10.0.1](https://pypi.org/project/ansible/10.0.1/))
- [git](https://git-scm.com/downloads)
- [jq](https://jqlang.github.io/jq/download/)
- [yq](https://lindevs.com/install-yq-on-ubuntu/)
- [jmespath](https://pypi.org/project/jmespath/)
- [ability to run sudo without password](https://linuxhandbook.com/sudo-without-password/)
- Install venv (if not already installed)
  ```bash
  sudo apt-get install python3-venv
  python3 -m venv ~/.venv
  source ~/.venv/bin/activate
  pip install ansible-core==2.18.1
  ```
## Base Infrastructure requirements

| Component      | OS              | CPU    | RAM      | Storage                          | EIP   |
|----------------|-----------------|--------|----------|----------------------------------|-------|
| bastion        | Ubuntu 22.04 LTS | 2vCPU  | 2GB     | 10GB                             | 1 EIP |
| ext-haproxy    | Ubuntu 22.04 LTS | 2vCPU  | 2GB     | 10GB                             | 1 EIP |
| int-haproxy    | Ubuntu 22.04 LTS | 2vCPU  | 2GB     | 10GB                             |       |
| microk8s1      | Ubuntu 22.04 LTS | 16vCPU  | 64GB   | 100GB + 500GB extra storage      |       |
| microk8s2      | Ubuntu 22.04 LTS | 16vCPU  | 64GB   | 100GB + 500GB extra storage      |       |
| microk8s3      | Ubuntu 22.04 LTS | 16vCPU  | 64GB   | 100GB + 500GB extra storage      |       |



## Create DNS zones in your AWS domain provider

AWS Hosted Zones Configuration
eg, you have example.com domain
You need to create two hosted zones like below;
Domain Name: int.cc.example.com
Domain Name: cc.example.com
These two hosted zones will manage the DNS records for your internal (private) and external (public) resources.

## Configuration

### Configuring the External HAProxy Proxy Node

#### Install HAProxy
Run the following commands to install HAProxy:

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get -y install haproxy
```

#### Update `/etc/haproxy/haproxy.cfg`
Modify the `/etc/haproxy/haproxy.cfg` file with the following configuration:

```yaml
defaults
        log     global
        mode    tcp
        option  tcplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

frontend external-k8s-tls
  mode tcp
  bind <ext-haproxy-private-ip>:443
  default_backend external-k8s-tls

frontend external-k8s
  mode tcp
  bind <ext-haproxy-private-ip>:80
  default_backend external-k8s

backend external-k8s-tls
  mode tcp
  balance roundrobin
  option ssl-hello-chk
    server node1 <microk8s-1-private-ip>:32443 send-proxy-v2-ssl
    server node2 <microk8s-2-private-ip>:32443 send-proxy-v2-ssl
    server node3 <microk8s-3-private-ip>:32443 send-proxy-v2-ssl

backend external-k8s
  mode tcp
  balance roundrobin
    server node1 <microk8s-1-private-ip>:32080
    server node2 <microk8s-2-private-ip>:32080
    server node3 <microk8s-3-private-ip>:32080
```

#### Test and Restart HAProxy
1. Test the configuration for errors:
   ```bash
   sudo haproxy -f /etc/haproxy/haproxy.cfg -c
   ```

2. Restart the HAProxy service:
   ```bash
   sudo systemctl restart haproxy
   ```

### Configuring the Internal HAProxy Proxy Node

#### Install HAProxy
Run the following commands to install HAProxy:

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get -y install haproxy
```

#### Update `/etc/haproxy/haproxy.cfg`
Modify the `/etc/haproxy/haproxy.cfg` file with the following configuration:

```yaml
defaults
        log     global
        mode    tcp
        option  tcplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

frontend internal-k8s-tls
  mode tcp
  bind <int-haproxy-private-ip>:443
  default_backend internal-k8s-tls

frontend internal-k8s
  mode tcp
  bind <int-haproxy-private-ip>:80
  default_backend internal-k8s

backend internal-k8s-tls
  mode tcp
  balance roundrobin
  option ssl-hello-chk
    server node1 <microk8s-1-private-ip>:31443
    server node2 <microk8s-2-private-ip>:31443
    server node3 <microk8s-3-private-ip>:31443

backend internal-k8s
  mode tcp
  balance roundrobin
    server node1 <microk8s-1-private-ip>:31080
    server node2 <microk8s-2-private-ip>:31080
    server node3 <microk8s-3-private-ip>:31080
```

#### Test and Restart HAProxy
1. Test the configuration for errors:
   ```bash
   sudo haproxy -f /etc/haproxy/haproxy.cfg -c
   ```

2. Restart the HAProxy service:
   ```bash
   sudo systemctl restart haproxy
   ```

## Getting Started
To get started with this Terraform codebase:
1. Clone the repository:
   ```bash
   git clone git@github.com:mojaloop/iac-modules.git
   ```

2. Navigate to the control center directory:
   ```bash
   cd iac-modules/terraform/ccnew
   ```
### Update custom configs
1. Update custom-config/cluster-config.yaml: Please make sure to update the vaules as per your requirements and don't use dummy vaules
   ```bash
   iac_terraform_modules_tag: v6.0.0-rc003
   ansible_collection_tag: v5.5.0-rc5
   cloud_platform: bare-metal
   cluster_name: dev
   coredns_bind_address: 169.254.20.10
   dns_provider: aws
   dns_zone_force_destroy: true
   domain: <example.com>
   enable_cloud_csi_provisioner: true
   enable_object_storage_backend: true
   iac_group_name: iac_admin
   k8s_cluster_module: base-k8s
   k8s_cluster_type: microk8s
   kubernetes_oidc_enabled: true
   letsencrypt_email: test@example.com
   master_node_supports_traffic: true
   microk8s_dev_skip: false
   object_storage_base_bucket_name: velero
   object_storage_force_destroy_bucket: true
   vpc_cidr: <10.0.0.0/16>
   ```

2. Update custom-config/cluster-config.yaml: Please make sure to update the vaules as per your requirements and don't use dummy vaules   
   ```bash
   external_load_balancer_dns: <ext-haproxy-publicip>
   wireguard_port: 31821
   dns_provider: aws
   create_ext_dns_user: true
   create_iam_user: true
   external_dns_cloud_role: "arn:ext-dns-cloud-role"
   ext_dns_cloud_policy: "arn:blah"
   object_storage_cloud_role: "arn:obj-store-role"
   backup_bucket_name: storage-bucket
   nat_public_ips: ["<natpublicip>"]
   internal_load_balancer_dns: <int-haproxy-privateip>
   egress_gateway_cidr: <10.0.0.0/16>
   bastion_public_ips: ["<bastion-publicip>"]
   private_subdomain: "<private-subdomain>"
   public_subdomain: "<public-subdomain>"
   int_interop_switch_subdomain: intapi
   ext_interop_switch_subdomain: extapi
   target_group_internal_https_port: 31443
   target_group_internal_http_port: 31080
   target_group_external_https_port: 32443
   target_group_external_http_port: 32080
   target_group_internal_health_port: 31081
   target_group_external_health_port: 32081
   private_network_cidrs:
     - <10.0.10.0/24>
   ssh_private_key: |
     -----BEGIN RSA PRIVATE KEY-----
   
     -----END RSA PRIVATE KEY-----
   
   os_user_name: ubuntu
   base_domain: "<example.com>"
   kubeapi_loadbalancer_fqdn: none
   master_hosts_0_private_ip: "<10.0.10.11>"
   agent_hosts: {}
   master_hosts:
     <hostname_microk8s_1>:  # This should be the hostname of the first MicroK8s node.
       ip: <10.0.10.11>
       node_taints: []
       node_labels:
         workload-class.mojaloop.io/CENTRAL-LEDGER-SVC: "enabled"
         workload-class.mojaloop.io/CORE-API-ADAPTERS: "enabled"
         workload-class.mojaloop.io/CENTRAL-SETTLEMENT: "enabled"
         workload-class.mojaloop.io/QUOTING-SERVICE: "enabled"
         workload-class.mojaloop.io/ACCOUNT-LOOKUP-SERVICE: "enabled"
         workload-class.mojaloop.io/ALS-ORACLES: "enabled"
         workload-class.mojaloop.io/CORE-HANDLERS: "enabled"
         workload-class.mojaloop.io/KAFKA-CONTROL-PLANE: "enabled"
         workload-class.mojaloop.io/KAFKA-DATA-PLANE: "enabled"
         workload-class.mojaloop.io/RDBMS-CENTRAL-LEDGER-LIVE: "enabled"
         workload-class.mojaloop.io/RDBMS-ALS-LIVE: "enabled"
     <hostname_microk8s-2>:
       ip: <10.0.10.12>
       node_taints: []
       node_labels:
         workload-class.mojaloop.io/CENTRAL-LEDGER-SVC: "enabled"
         workload-class.mojaloop.io/CORE-API-ADAPTERS: "enabled"
         workload-class.mojaloop.io/CENTRAL-SETTLEMENT: "enabled"
         workload-class.mojaloop.io/QUOTING-SERVICE: "enabled"
         workload-class.mojaloop.io/ACCOUNT-LOOKUP-SERVICE: "enabled"
         workload-class.mojaloop.io/ALS-ORACLES: "enabled"
         workload-class.mojaloop.io/CORE-HANDLERS: "enabled"
         workload-class.mojaloop.io/KAFKA-CONTROL-PLANE: "enabled"
         workload-class.mojaloop.io/KAFKA-DATA-PLANE: "enabled"
         workload-class.mojaloop.io/RDBMS-CENTRAL-LEDGER-LIVE: "enabled"
         workload-class.mojaloop.io/RDBMS-ALS-LIVE: "enabled"
     <hostname_microk8s-3>:
       ip: <10.0.10.13>
       node_taints: []
       node_labels:
         workload-class.mojaloop.io/CENTRAL-LEDGER-SVC: "enabled"
         workload-class.mojaloop.io/CORE-API-ADAPTERS: "enabled"
         workload-class.mojaloop.io/CENTRAL-SETTLEMENT: "enabled"
         workload-class.mojaloop.io/QUOTING-SERVICE: "enabled"
         workload-class.mojaloop.io/ACCOUNT-LOOKUP-SERVICE: "enabled"
         workload-class.mojaloop.io/ALS-ORACLES: "enabled"
         workload-class.mojaloop.io/CORE-HANDLERS: "enabled"
         workload-class.mojaloop.io/KAFKA-CONTROL-PLANE: "enabled"
         workload-class.mojaloop.io/KAFKA-DATA-PLANE: "enabled"
         workload-class.mojaloop.io/RDBMS-CENTRAL-LEDGER-LIVE: "enabled"
         workload-class.mojaloop.io/RDBMS-ALS-LIVE: "enabled"
   k6s_callback_fqdn: none
   enable_k6s_test_harness: false
   test_harness_private_ip: none
   dns_resolver_ip: 8.8.8.8
   rook_disk_vol: "/dev/sdb"
   ci_iam_user_access_key: "<aws_access_key>"
   ci_iam_user_secret_key: "<aws_secret_key>"
   ci_iam_user_client_secret_name: "AWS_SECRET_ACCESS_KEY"
   ci_iam_user_client_id_name: "AWS_ACCESS_KEY_ID"
   route53_external_dns_access_key: "<aws_access_key>"
   route53_external_dns_secret_key: "<aws_secret_key>"
   external_dns_credentials_client_id_name: "AWS_ACCESS_KEY_ID"
   external_dns_credentials_client_secret_name: "AWS_SECRET_ACCESS_KEY"
   cert_manager_credentials_client_id_name: "AWS_ACCESS_KEY_ID"
   cert_manager_credentials_client_secret_name: "AWS_SECRET_ACCESS_KEY"
   ```

3. Update custom-config/common-vars.yaml: Please make sure to update the vaules as per your requirements and don't use dummy vaules

   ```bash
   rook_ceph_helm_version: "1.15.5"
   rook_ceph_image_version: "v18.2.4"
   rook_ceph_mon_volumes_size: "10Gi"
   rook_ceph_osd_volumes_storage_class: "microk8s-hostpath"
   rook_ceph_mon_volumes_storage_class: "microk8s-hostpath"
   rook_ceph_cloud_pv_reclaim_policy: "Delete" # Retain, Delete
   rook_ceph_csi_driver_replicas: "'2'"
   rook_ceph_objects_replica_count: "'1'"
   rook_ceph_osd_count: "'3'"
   rook_ceph_volume_size_per_osd: "300Gi"
   rook_ceph_volumes_provider: "host" # host, pvc
   rook_ceph_aws_ebs_csi_driver_helm_version: "2.39.0"
   ```   

4. Update custom-config/environment.yaml:
   ```bash
   environments:
      - hub
      - pm4ml
   ```   

5. Set environment variables:
   ```bash
   source ~/.venv/bin/activate
   source ./externalrunner.sh
   source ./scripts/setlocalvars.sh
   ```   

## Deployment
To apply the configurations and deploy the Mojaloop Control Center on your infrastructure, follow these steps:
1. Initialize the Deployment
Start by planning the deployment to understand the changes that will be made. This step initializes Terraform and prepares it for provisioning:
   ```bash
   terragrunt run-all init
   ```

2. Apply the Changes
Once the plan is initialized, apply the changes to your infrastructure. This step will start provisioning your microk8s cluster and deploy the necessary resources.
If this process is taking too long and you would like to monitor the progress, you can move on to Step 3 via different termianl and start checking the deployment status while the terragrunt provisioning continues in the background.
   ```bash
   terragrunt run-all apply --terragrunt-non-interactive 
   ```
4. Monitor the Deployment Progress
The provisioning process may take some time. To monitor the status of your deployments, you can SSH into your Bastion Host and check the status of the applications deployed by ArgoCD.
 ```bash
   ssh -i <sshkey> ubuntu@<bastionpublicip>
   sudo su -
   ls ~/.kube/
   export KUBECONFIG=~/.kube/kubeconfig
   
   ### Install and Use k9s for Cluster Management (Optional)
   wget https://github.com/derailed/k9s/releases/download/v0.50.4/k9s_linux_amd64.deb 
   dpkg -i k9s_linux_amd64.deb
   k9s
   
   ### Alternatively, you can use kubectl to check the status of the applications. Keep in mind that it may take some time for all applications to sync and reach the "Synced" state.
   kubectl get app -n argocd
   ### You should see output similar to the following, indicating that each application has been synced and is healthy:
   NAME                          SYNC STATUS   HEALTH STATUS
   argocd                        Synced        Healthy
   base-monitoring               Synced        Healthy
   cert-manager                  Synced        Healthy
   crossplane                    Synced        Healthy
   deploy-env-config             Synced        Healthy
   dns-utils                     Synced        Healthy
   dns-utils-post-config         Synced        Healthy
   dns-utils-pre                 Synced        Healthy
   external-secrets              Synced        Healthy
   gitlab                        Synced        Healthy
   gitlab-post-config            Synced        Healthy
   gitlab-pre                    Synced        Healthy
   istio                         Synced        Healthy
   istio-gw                      Synced        Healthy
   k8s-post-config               Synced        Healthy
   kyverno                       Synced        Healthy
   maintenance-pre               Synced        Healthy
   monitoring                    Synced        Healthy
   monitoring-post-config        Synced        Healthy
   monitoring-pre                Synced        Healthy
   netbird                       Synced        Healthy
   netbird-post-config           Synced        Healthy
   netbird-pre                   Synced        Healthy
   nexus                         Synced        Healthy
   nexus-post-config             Synced        Healthy
   nexus-pre                     Synced        Healthy
   percona-postgresql-operator   Synced        Healthy
   redis-operator                Synced        Healthy
   reflector                     Synced        Healthy
   reloader                      Synced        Healthy
   rook-ceph                     Synced        Healthy
   root-deployer                 Synced        Healthy
   utils-post-config             Synced        Healthy
   vault                         Synced        Healthy
   vault-post-config             Synced        Healthy
   velero                        Synced        Healthy
   xplane-db-provider            Synced        Healthy
   zitadel                       Synced        Healthy
   zitadel-post-config           Synced        Healthy
   zitadel-pre                   Synced        Healthy
   ```

4. After all the applications are up and running in the cluster you can start moving terraform state to the microk8s
   ```bash
   ./movestatetok8s.sh
   ```
   
5. Login to the Gitlab with credentials from the gitlab secrets in the gitlab namespace. Then,please make sure to setup 2FA for root user
6. Once you able to login Gitlab, you'll see repository name: bootstrap which is also know as control center repository
7. Go to the CI/CD pipeline and run the **deploy** job. 
8. Afterward, create a CI/CD variable named **ENV_TO_UPDATE** and **IAC_MODULES_VERSION_TO_UPDATE** in the bootstrap CI/CD settings
eg,
```bash
ENV_TO_UPDATE : hub	
IAC_MODULES_VERSION_TO_UPDATE : v6.0.0-rc003
```	
9. Finally, run the **deploy-env-templates** job. Afterward, you will see that your environment repository has been created in GitLab.


## Best Practices
- Always write clear and concise comments in your code whenever you commit to the GitLab repositories.
- Avoid canceling jobs once they start running in GitLab CI. Let them fail naturally if necessary.
- Regularly update your Terraform provider plugins, Terraform version, and application versions as needed for your requirements.

## Troubleshooting
- Check the output of terragrunt output and terragrunt apply for error messages related to GitLab.
- Ensure that your provided credentials or vaules are correctly configured.
- Review logs and state files for inconsistencies.
- Check the ArgoCD sync status to ensure your applications are correctly deployed.
```bash
kubectl get app -n argocd
```
- If you need to unlock terraform state in Kubernetes, follow these steps:
```bash
kubectl get lease -A 
kubectl delete lease <lock-tfstate-xxx-state>
```

