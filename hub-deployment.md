# On-Premise Deployment of Mojaloop Hub

[![GitHub Issues](https://img.shields.io/github/issues/mojaloop/iac-modules)](https://github.com/mojaloop/iac-modules/issues)
[![GitHub Stars](https://img.shields.io/github/stars/mojaloop/iac-modules)](https://github.com/mojaloop/iac-modules)

This repository provides a step-by-step guide for deploying the **Mojaloop Hub** in an on-premise environment. While Mojaloop is typically cloud-based for scalability and security, on-premise deployments are ideal for compliance with data residency laws, reduced latency, or enhanced infrastructure control. This guide is tailored for DevOps engineers and system administrators experienced with Kubernetes, Terraform, and Ansible.

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

The **Mojaloop Hub** is a critical component of the Mojaloop platform, enabling interoperable digital financial services to promote financial inclusion. This guide details the process of deploying the Hub on-premise, covering infrastructure setup, configuration, and deployment using modern DevOps tools.

---

## 🛠 Key Technical Requirements

To deploy the Mojaloop Hub, ensure proficiency in the following:

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

##Control Center must be deployed first.

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
- `int.hub.example.com` (internal/private resources)
- `hub.example.com` (external/public resources)

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

Open the hub repository and navigate to Gitlab Web IDE:

---
### Custom Configuration Files

Update the following files in `custom-config/`:

1. **`addons-vars.yaml`**: (Optional)
   ```yaml
   addons_github_org: mojaloop
   addons_github_repo: iac-modules
   addons_github_module_path: terraform/gitops/k8s-addons-config
   addons_github_module_tag: v6.0.0-rc003
   ```

1. **`cluster-config.yaml`**:
   ```yaml
   env: hub
   vpc_cidr: <10.10.0.0/16>
   managed_vpc_cidr: <10.130.0.0/16>
   domain: <example.com>
   currency: <EUR>
   managed_svc_enabled: false
   k8s_cluster_type: microk8s
   cloud_platform: bare-metal
   iac_terraform_modules_tag: v6.0.0-rc003
   ansible_collection_tag: v5.5.0-rc5
   coredns_bind_address: 169.254.20.10
   dns_provider: aws
   dns_zone_force_destroy: true
   enable_cloud_csi_provisioner: true
   enable_object_storage_backend: true
   iac_group_name: iac_admin
   k8s_cluster_module: base-k8s
   kubernetes_oidc_enabled: true
   letsencrypt_email: <test@example.com>
   master_node_supports_traffic: true
   microk8s_dev_skip: false
   object_storage_base_bucket_name: velero
   object_storage_force_destroy_bucket: true
   
   ```

2. **`bare-metal-vars.yaml`**:

   ```yaml
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
   egress_gateway_cidr: <10.0.0.0/16>
   bastion_public_ips: ["<bastion-publicip>"]
   private_subdomain: "<int.hub.example.com>"
   public_subdomain: "<hub.example.com>"
   int_interop_switch_subdomain: intapi
   ext_interop_switch_subdomain: extapi
   target_group_internal_https_port: 31443
   target_group_internal_http_port: 31080
   target_group_external_https_port: 32443
   target_group_external_http_port: 32080
   target_group_internal_health_port: 31081
   target_group_external_health_port: 32081
   private_network_cidrs:
     - <10.110.0.0/16>
   ssh_private_key: |
     -----BEGIN RSA PRIVATE KEY-----
     -----END RSA PRIVATE KEY-----
   os_user_name: ubuntu
   base_domain: "<hub.example.com>"
   kubeapi_loadbalancer_fqdn: none
   master_hosts_0_private_ip: "<10.110.10.3>"
   agent_hosts: {}
   master_hosts:
     <dev-hub-microk8s-1>:
       ip: <10.110.10.3>
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
         workload-class.mojaloop.io/MONITORING: "enabled"
     <dev-hub-microk8s-2>:
       ip: <10.110.10.5>
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
         workload-class.mojaloop.io/MONITORING: "enabled"
    < dev-hub-microk8s-3>:
       ip: <10.110.10.2>
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
         workload-class.mojaloop.io/MONITORING: "enabled"
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

3. **`common-vars.yaml`**:

   ```yaml
   longhorn_backup_job_enabled: false
   ```

4. **`finance-portal-values-override.yaml`**
   ```yaml
   reporting-events-processor-svc:
     replicaCount: 1
   reporting-hub-bop-settlements-ui:
     image:
       tag: v0.0.22   
   reporting-legacy-api:
     install-templates: false
     auth: false
   ```

5. **`mcm-values-override.yaml`**
   ```yaml
   api:
      image:
       name: ghcr.io/infitx/connection-manager-api
       version:  v2.9.6
   ui:
     image:
       version: 1.8.4
   ```

6. **`mojaloop-vars.yaml`**
   ```yaml
   mojaloop_chart_version: 17.0.0
   finance_portal_chart_version: 5.0.1
   central_ledger_handler_transfer_position_batch_processing_enabled: true
   reporting_templates_chart_version: 1.1.15
   ml_testing_toolkit_cli_chart_version: 15.9.0
   currency: ${currency}
   mcm_chart_version: 1.2.10
   onboarding_net_debit_cap: 1000
   onboarding_funds_in: 1000
   ``` 
7. **`values-hub-provisioning-override.yaml`**
   ```yaml
   testCaseEnvironmentFile:
     inputValues:
       DEFAULT_SETTLEMENT_MODEL_NAME: DEFERREDNET
   ```    

8. **`.gitlab-ci.yml`**: Update Line 314 to skip lint-apps job for the very firt time
   ```yaml
   if [ "$(./bin/yq eval .cloud_platform ../../$CONFIG_PATH/cluster-config.yaml)" = "bare-metal" ]; then echo "Skipping linting when not bare metal"; exit 0; fi
   ```
    


---

## 🚀 Deployment
1. **Gitlab CI**:
Navigate to the CI/CD pipeline section of your Mojaloop Hub repository in GitLab. Once there, locate the `deploy-infra` job, which is responsible for deploying your infrastructure. 
Trigger this job to initiate the deployment process. This will automatically begin provisioning the necessary resources and configure the infrastructure according to the specifications defined in your repository's configuration files. Make sure that the pipeline is successfully executed, and monitor for any potential errors or issues that may arise during deployment.


2. **Monitor Deployment**:

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

