# On-Premise Deployment of PM4ML

[![GitHub Issues](https://img.shields.io/github/issues/mojaloop/iac-modules)](https://github.com/mojaloop/iac-modules/issues)
[![GitHub Stars](https://img.shields.io/github/stars/mojaloop/iac-modules)](https://github.com/mojaloop/iac-modules)

This repository provides a step-by-step guide for deploying the **Payment Manager (PM4ML)** in an on-premise environment. This guide is tailored for DevOps engineers and system administrators experienced with Kubernetes, Terraform, and Ansible.

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

This guide details the process of deploying the PM4ML on-premise, covering infrastructure setup, configuration, and deployment using modern DevOps tools.

---

## 🛠 Key Technical Requirements

To deploy the PM4ML, ensure proficiency in the following:

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
| MicroK8s Node 1 | Ubuntu 22.04 LTS | 16 vCPU | 32 GB | 100 GB + 500 GB extra       |       |
| MicroK8s Node 2 | Ubuntu 22.04 LTS | 16 vCPU | 32 GB | 100 GB + 500 GB extra       |       |
| MicroK8s Node 3 | Ubuntu 22.04 LTS | 16 vCPU | 32 GB | 100 GB + 500 GB extra       |       |

### DNS Configuration

Create two AWS Route 53 hosted zones for your domain (e.g., `example.com`):
- `int.pm4ml.example.com` (internal/private resources)
- `pm4ml.example.com` (external/public resources)

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
   env: pm4ml
   vpc_cidr: <10.120.0.0/16>
   managed_vpc_cidr: 10.130.0.0/16
   domain: <example.com>
   managed_svc_enabled: false
   k8s_cluster_type: microk8s
   cloud_platform: bare-metal
   ansible_collection_tag: v5.5.0-rc5
   iac_terraform_modules_tag: v6.0.0-rc003   
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
   private_subdomain: "<int.pm4ml.example.com>"
   public_subdomain: "<pm4ml.example.com>"
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
   base_domain: "<pm4ml.example.com>"
   kubeapi_loadbalancer_fqdn: none
   master_hosts_0_private_ip: "<10.110.10.3>"
   agent_hosts: {}
   master_hosts:
     <dev-hub-microk8s-1>:
       ip: <10.110.10.3>
       node_taints: []
       node_labels:
         workload-class.mojaloop.io/MONITORING: "enabled"
     <dev-hub-microk8s-2>:
       ip: <10.110.10.5>
       node_taints: []
       node_labels:
         workload-class.mojaloop.io/MONITORING: "enabled"
    < dev-hub-microk8s-3>:
       ip: <10.110.10.2>
       node_taints: []
       node_labels:
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
   mcm_enabled: false
   mojaloop_enabled: false
   pm4ml_enabled: true
   ```

4. **`pm4ml-values-override.yaml`**
   ```yaml
   management-api:
     image:
       tag: 6.7.0
     env:
      ENABLE_UI_API_SERVER : true
   
   experience-api:
     image:
       tag: 3.1.2-snapshot.5
   ```

5. **`pm4ml-values`**
   ```yaml
   pm4mls:
      dfsp1:
        domain: ${replace(env,"/pm$/","")}.${domain}
        pm4ml_enabled: true
        pm4ml_chart_version: 10.2.0
        pm4ml_external_mcm_public_fqdn: mcm.<hub.example.com>
        pm4ml_ingress_internal_lb: false
        pm4ml_external_switch_client_id: dfsp-jwt
        pm4ml_external_switch_oidc_url: https://keycloak.<hub.example.com>
        pm4ml_external_switch_oidc_token_route: realms/dfsps/protocol/openid-connect/token
        pm4ml_external_switch_client_secret_vault_path: "mcmdev_client_secret"
        pm4ml_external_switch_fqdn: extapi.<hub.example.com>
        pm4ml_dfsp_id: dfsp1
        pm4ml_ttk_enabled: false
        auto_accept_party: false
        enable_sdk_bulk_transaction_support: false
        opentelemetry_enabled: false
        opentelemetry_namespace_filtering_enable: false
        ## core_connector_selected can be one of the following values: "ttk", "cc", "custom"
        core_connector_selected: custom
        ## custom_core_connector_endpoint is only required if core_connector_selected is set to "custom"
        custom_core_connector_endpoint: dfsp1-mojaloop-core-connector:3003
        pm4ml_reserve_notification: false
        currency: <EUR>
        core_connector_config:
          image:
            repository: <yourcore-connector-repositoryurl>
            tag: <1.0.0>
        payment_token_adapter_config: {
          enabled: false
        }
        ui_custom_config:
          TITLE: Payment Manager

   ```

6. **`stateful-resources-operators.yaml`**
   ```yaml
   strimzi:
     enabled: false
     install_type: helm
     helm_chart: strimzi-kafka-operator
     namespace: strimzi
     release_name: strimzi
     helm_chart_version: 0.40.0
     helm_chart_repo: https://strimzi.io/charts
     helm_chart_values_file: values-strimzi.yaml
   percona-mysql:
     enabled: true
     install_type: helm
     helm_chart: pxc-operator
     namespace: percona-mysql
     release_name: percona-mysql
     helm_chart_version: 1.14.0
     helm_chart_repo: https://percona.github.io/percona-helm-charts/
     helm_chart_values_file: values-percona-mysql.yaml
   percona-mongodb:
     enabled: false
     install_type: helm
     helm_chart: psmdb-operator
     namespace: percona-mongodb
     release_name: percona-mongodb
     helm_chart_version: 1.16.2
     helm_chart_repo: https://percona.github.io/percona-helm-charts/
     helm_chart_values_file: values-percona-mongodb.yaml
   redis:
     enabled: false
     install_type: operator
     helm_chart: redis-operator
     namespace: redis
     release_name: redis
     helm_chart_version: 0.20.0
     helm_chart_repo: https://ot-container-kit.github.io/helm-charts/
     helm_chart_values_file: values-redis.yaml
   ``` 

7. **`.gitlab-ci.yml`**: Update Line 314 to skip lint-apps job for the very firt time
   ```yaml
   if [ "$(./bin/yq eval .cloud_platform ../../$CONFIG_PATH/cluster-config.yaml)" = "bare-metal" ]; then echo "Skipping linting when not bare metal"; exit 0; fi
   ```
    


---

## 🚀 Deployment
1. **Gitlab CI**:
Navigate to the CI/CD pipeline section of your PM4ML repository in GitLab. Once there, locate the `deploy-infra` job, which is responsible for deploying your infrastructure. 
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

