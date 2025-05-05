

# Mojaloop Control Center On-Premise Deployment

[![GitHub Issues](https://img.shields.io/github/issues/mojaloop/iac-modules)](https://github.com/mojaloop/iac-modules/issues)
[![GitHub Stars](https://img.shields.io/github/stars/mojaloop/iac-modules)](https://github.com/mojaloop/iac-modules)

This repository provides a step-by-step guide for deploying the **Mojaloop Control Center** in an on-premise environment. While Mojaloop is typically cloud-based for scalability and security, on-premise deployments are ideal for compliance with data residency laws, reduced latency, or enhanced infrastructure control. This guide is tailored for DevOps engineers and system administrators experienced with Kubernetes, Terraform, and Ansible.

---

## 📋 Table of Contents

- [Introduction](#introduction)
- [Key Technical Requirements](#key-technical-requirements)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
---

## 📖 Introduction

The **Mojaloop Control Center** is a critical component of the Mojaloop platform, enabling interoperable digital financial services to promote financial inclusion. This guide details the process of deploying the Control Center on-premise, covering infrastructure setup, configuration, and deployment using modern DevOps tools.

---

## 🛠 Key Technical Requirements

To deploy the Mojaloop Control Center, ensure proficiency in the following:

- **Infrastructure**: Knowledge of cloud and on-premise deployment models.
- **Terraform/Terragrunt**: Infrastructure provisioning with modular, DRY configurations.
- **Ansible**: Automation of configuration and deployment tasks.
- **Docker/Kubernetes**: Containerization and orchestration of microservices.
- **Git**: Version control for repository management.
- **Bash**: Scripting for automation.
- **Networking**: Firewalls, load balancers, DNS, and NAT gateways.
- **Security**: Compliance with data residency and security best practices.
- **Monitoring**: Tools like Grafana and Loki for observability.
- **Zitadel**: Open-source IAM for authentication and SSO.
- **NetBird**: Zero Trust Network Access (ZTNA) for secure networking.
- **Istio**: Familiarity with Istio service mesh for managing microservices communication, traffic routing, security (mTLS), and observability in Kubernetes clusters.

---

## ✅ Prerequisites

Install the following tools:

| Tool          | Version       | Link |
|---------------|---------------|------|
| Terraform     | v1.3.2        | [Download](https://www.terraform.io/downloads.html) |
| Terragrunt    | v0.57.0       | [Download](https://github.com/gruntwork-io/terragrunt/releases) |
| Python 3      | v3.12         | [Download](https://www.python.org/downloads/) |
| Ansible       | v2.18.1 / v10.0.1 | [Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) |
| Git           | Latest        | [Download](https://git-scm.com/downloads) |
| jq            | Latest        | [Download](https://jqlang.github.io/jq/download/) |
| yq            | Latest        | [Installation Guide](https://lindevs.com/install-yq-on-ubuntu/) |
| jmespath      | Latest        | `pip install jmespath` |
| sudo (no password) | -         | [Guide](https://linuxhandbook.com/sudo-without-password/) |

Set up a Python virtual environment:

```bash
sudo apt-get install python3-venv
python3 -m venv ~/.venv
source ~/.venv/bin/activate
pip install ansible-core==2.18.1
```

### Infrastructure Requirements

| Component       | OS               | CPU     | RAM  | Storage                     | EIP   |
|----------------|------------------|---------|------|-----------------------------|-------|
| Bastion        | Ubuntu 22.04 LTS | 2 vCPU  | 2 GB | 10 GB                       | 1 EIP |
| External HAProxy | Ubuntu 22.04 LTS | 2 vCPU  | 2 GB | 10 GB                       | 1 EIP |
| Internal HAProxy | Ubuntu 22.04 LTS | 2 vCPU  | 2 GB | 10 GB                       |       |
| MicroK8s Node 1 | Ubuntu 22.04 LTS | 16 vCPU | 64 GB | 100 GB + 500 GB extra       |       |
| MicroK8s Node 2 | Ubuntu 22.04 LTS | 16 vCPU | 64 GB | 100 GB + 500 GB extra       |       |
| MicroK8s Node 3 | Ubuntu 22.04 LTS | 16 vCPU | 64 GB | 100 GB + 500 GB extra       |       |

### DNS Configuration

Create two AWS Route 53 hosted zones for your domain (e.g., `example.com`):
- `int.cc.example.com` (internal/private resources)
- `cc.example.com` (external/public resources)

---

## ⚙️ Configuration

### External HAProxy Node

1. **Install HAProxy**:

   ```bash
   sudo apt-get update
   sudo apt-get upgrade
   sudo apt-get -y install haproxy
   ```

2. **Configure HAProxy**:

   Update `/etc/haproxy/haproxy.cfg`:

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

3. **Test and Restart**:

   ```bash
   sudo haproxy -f /etc/haproxy/haproxy.cfg -c
   sudo systemctl restart haproxy
   ```

### Internal HAProxy Node

1. **Install HAProxy**:

   ```bash
   sudo apt-get update
   sudo apt-get upgrade
   sudo apt-get -y install haproxy
   ```

2. **Configure HAProxy**:

   Update `/etc/haproxy/haproxy.cfg`:

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

3. **Test and Restart**:

   ```bash
   sudo haproxy -f /etc/haproxy/haproxy.cfg -c
   sudo systemctl restart haproxy
   ```

---

## 🚀 Getting Started

Clone the repository and navigate to the Control Center directory:

```bash
git clone git@github.com:mojaloop/iac-modules.git
cd iac-modules/terraform/ccnew
```

---
### Custom Configuration Files

Update the following files in `custom-config/`:

1. **`cluster-config.yaml`**:

   ```yaml
   iac_terraform_modules_tag: v6.0.0-rc003
   ansible_collection_tag: v5.5.0-rc5
   cloud_platform: bare-metal
   cluster_name: <dev>
   coredns_bind_address: 169.254.20.10
   dns_provider: aws
   dns_zone_force_destroy: true
   domain: example.com
   enable_cloud_csi_provisioner: true
   enable_object_storage_backend: true
   iac_group_name: iac_admin
   k8s_cluster_module: base-k8s
   k8s_cluster_type: microk8s
   kubernetes_oidc_enabled: true
   letsencrypt_email: <test@example.com>
   master_node_supports_traffic: true
   microk8s_dev_skip: false
   object_storage_base_bucket_name: velero
   object_storage_force_destroy_bucket: true
   vpc_cidr: <10.0.0.0/16>
   external_load_balancer_dns: <ext-haproxy-publicip>
   wireguard_port: 31821
   create_ext_dns_user: true
   create_iam_user: true
   external_dns_cloud_role: "arn:ext-dns-cloud-role"
   ext_dns_cloud_policy: "arn:blah"
   object_storage_cloud_role: "arn:obj-store-role"
   backup_bucket_name: storage-bucket
   nat_public_ips: ["<natpublicip>"]
   internal_load_balancer_dns: <int-haproxy-privateip>
   egress_gateway_cidr: 10.0.0.0/16
   bastion_public_ips: ["<bastion-publicip>"]
   private_subdomain: "int.cc.example.com"
   public_subdomain: "cc.example.com"
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
     <your-ssh-private-key>
     -----END RSA PRIVATE KEY-----
   os_user_name: ubuntu
   base_domain: "example.com"
   kubeapi_loadbalancer_fqdn: none
   master_hosts_0_private_ip: "<10.0.10.11>"
   agent_hosts: {}
   master_hosts:
     <microk8s-1>:
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
     <microk8s-2>:
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
     <microk8s-3>:
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
   dns_resolver_ip: <8.8.8.8>
   rook_disk_vol: "</dev/sdb>"
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

2. **`common-vars.yaml`**:

   ```yaml
   rook_ceph_helm_version: "1.15.5"
   rook_ceph_image_version: "v18.2.4"
   rook_ceph_mon_volumes_size: "10Gi"
   rook_ceph_osd_volumes_storage_class: "microk8s-hostpath"
   rook_ceph_mon_volumes_storage_class: "microk8s-hostpath"
   rook_ceph_cloud_pv_reclaim_policy: "Delete"
   rook_ceph_csi_driver_replicas: "'2'"
   rook_ceph_objects_replica_count: "'1'"
   rook_ceph_osd_count: "'3'"
   rook_ceph_volume_size_per_osd: "300Gi"
   rook_ceph_volumes_provider: "host"
   rook_ceph_aws_ebs_csi_driver_helm_version: "2.39.0"
   ```

3. **`environment.yaml`**:

   ```yaml
   environments:
     - hub
     - pm4ml
   ```

4. **Set Environment Variables**:

   ```bash
   source ~/.venv/bin/activate
   source ./externalrunner.sh
   source ./scripts/setlocalvars.sh
   ```

---

## 🚀 Deployment

1. **Initialize Terraform**:

   ```bash
   terragrunt run-all init
   ```

2. **Apply Changes**:

   ```bash
   terragrunt run-all apply --terragrunt-non-interactive
   ```

3. **Monitor Deployment**:

   SSH into the Bastion host to monitor ArgoCD applications:

   ```bash
   ssh -i <sshkey> ubuntu@<bastionpublicip>
   sudo su -
   export KUBECONFIG=~/.kube/kubeconfig
   kubectl get app -n argocd
   ```

   Optionally, install `k9s` for cluster management:

   ```bash
   wget https://github.com/derailed/k9s/releases/download/v0.50.4/k9s_linux_amd64.deb
   dpkg -i k9s_linux_amd64.deb
   k9s
   ```

4. **Move Terraform State**:

   ```bash
   ./movestatetok8s.sh
   ```

5. **Configure GitLab**:

   - Log in to GitLab using credentials from the `gitlab` namespace secrets.
   - Enable 2FA for the root user.
   - In the `bootstrap` repository, run the `deploy` job in the CI/CD pipeline.
   - Add CI/CD variables:

     ```bash
     ENV_TO_UPDATE: hub
     IAC_MODULES_VERSION_TO_UPDATE: v6.0.0-rc003
     ```

   - Run the `deploy-env-templates` job to create the environment repository.

---

## 📝 Best Practices

- Write clear, concise comments in code commits.
- Allow GitLab CI jobs to complete or fail naturally; avoid canceling.
- Regularly update Terraform, Terragrunt, and application versions.

---

## 🛠 Troubleshooting

- Check `terragrunt` output and Gitlab CI jobs for errors.
- Verify configuration values and credentials.
- Monitor ArgoCD sync status:

  ```bash
  kubectl get app -n argocd
  ```

- Unlock Terraform state if needed:

  ```bash
  kubectl get lease -A
  kubectl delete lease <lock-tfstate-xxx-state>
  ```

