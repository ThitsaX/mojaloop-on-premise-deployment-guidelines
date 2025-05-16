# EXPERIMENTAL: Mojaloop On-Premise Deployment Guidelines
Mojaloop deployments are predominantly cloud-based due to their scalability, cost-effectiveness, and inherent security. However, there are instances where organizations opt for a self-hosted on-premise solution due to budget constraints or the readiness of alternative solutions. On-premise setups offer several benefits, including enhanced transparency regarding system limitations and costs, clear ownership of issues, improved visibility into system performance, reduced latency, and the necessity for compliance with regulations that require data to be stored within the country's own data center.

This document provides an overview of the Mojaloop Control Center and Hub deployments, focusing on its deployment, particularly in on-premise environments. Understanding the deployment process is essential for organizations looking to leverage Mojaloop for enhancing financial inclusion through interoperable digital financial services.
## Table of Contents
1. [Key Technical Requirements](#key-technical-requirements)
2. [Deploy Control Center] (#deploy-control-center)
3. [Deploy Hub Environment](#deploy-mojaloop-hub)

## Key Technical Requirements 
1. Infrastructure:
    Understanding of cloud computing principles, especially regarding scalability, cost-effectiveness, and security.
    Familiarity with both cloud-based and on-premise deployment models, including their respective advantages and limitations.

2. Terraform and Terragrunt:
    Proficiency in using Terraform for provisioning and managing infrastructure.
    Knowledge of writing modular and reusable Terraform configurations to facilitate efficient resource deployment.   
    Understanding what Terragrunt is: a thin wrapper for Terraform that provides extra tools for keeping your configurations DRY (Don't Repeat Yourself).
    Familiarity with how Terragrunt helps manage Terraform configurations across multiple environments and modules.

4. Ansible:
    Experience with Ansible for configuration management and automation of deployment tasks.
    Ability to create playbooks and manage inventory files for orchestrating software installations and updates.

5. Containerization and Orchestration:
    Understanding of Docker for containerization and Kubernetes (K8s) for orchestration.
    Familiarity with deploying applications in a microservices architecture using K8s.

6. Version Control Systems:
    Proficient use of Git for version control, including branching, merging, and managing repositories.

7. Scripting Languages:
    Basic knowledge of shell scripting (e.g., Bash) to automate tasks within the deployment process.

8. Installation and Configuration:
    Ability to install necessary tools such as Terraform, Ansible, Git, jq, and yq.
    Experience in configuring environments through YAML files and other configuration management tools.

9. Troubleshooting:
    Skills in diagnosing issues during deployment processes and applying best practices for troubleshooting common problems.

10. Networking:
    Understanding of networking concepts relevant to deployments, including security groups or firewalls, load balancers, nat gateways and domain management.

11. Compliance and Security:
    Awareness of regulatory requirements regarding data storage and processing within specific jurisdictions.
    Knowledge of security best practices for managing sensitive data.

12. Monitoring and Observability:
    Familiarity with tools for monitoring application performance (e.g., Grafana) and logging (e.g., Loki).
    Ability to implement observability solutions that provide insights into system performance.



## Deploy Control Center
To continue with the deployment of the Control Center environment, please follow the detailed guidelines provided in the [Control Center Deployment Documentation](https://github.com/ThitsaX/mojaloop-on-premise-deployment-guidelines/blob/main/control-center-deployment.md).


## Deploy Mojaloop HUB
To continue with the deployment of the Hub environment, please follow the detailed guidelines provided in the [Hub Deployment Documentation](https://github.com/ThitsaX/mojaloop-on-premise-deployment-guidelines/blob/main/hub-deployment.md).