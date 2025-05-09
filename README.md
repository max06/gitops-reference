# GitOps Repository Schema Documentation

This document outlines the structure and usage of a Git repository designed for GitOps, managing application deployments across multiple Kubernetes clusters using reusable templates. The schema supports a large number of clusters, optional grouping into "constellations," and variable inheritance for flexible configuration. It is tool-agnostic (e.g., compatible with Argo CD or similar tools) and focuses on the Git repository layout.

## Purpose
This repository serves as the single source of truth for:
- **Application Templates**: Reusable artifacts (Helm charts, Kustomize configurations, or plain manifests) for deploying applications.
- **Cluster Configurations**: Definitions of Kubernetes clusters, optionally grouped into constellations, with specific applications and parameters.
- **Variable Inheritance**: Shared and cluster-specific variables for parameterization, supporting interpolation in application configurations.

## Repository Structure
The repository is organized as follows:

```
.
├── templates/
│   └── <application-template-name>/
│       └── <template-files> (e.g., Chart.yaml, kustomization.yaml, manifests)
└── environments/
    └── constellations/
        ├── <constellation-name>/
        │   ├── constellation.yaml
        │   ├── applications/
        │   │   ├── <app-name>.yaml
        │   │   └── ...
        │   └── clusters/
        │       ├── <cluster-name>/
        │       │   ├── cluster.yaml
        │       │   └── applications/
        │       │       ├── <app-name>.yaml
        │       │       └── ...
        │       └── ...
        ├── standalone-clusters/
        │   ├── <cluster-name>/
        │   │   ├── cluster.yaml
        │   │   └── applications/
        │   │       ├── <app-name>.yaml
        │   │       └── ...
        │   └── ...
        └── ...
```

### Directory Breakdown
1. **templates/**
   - Contains reusable application templates.
   - Each subdirectory represents one template, named descriptively (e.g., `nginx-helm`, `my-app-kustomize`).
   - Templates can include:
     - Self-written Helm charts (with `Chart.yaml`, `values.yaml`, and `templates/`).
     - Public Helm charts with custom overrides.
     - Kustomize configurations (with `kustomization.yaml` and resources).
     - Plain Kubernetes manifests (e.g., `deployment.yaml`, `service.yaml`).
   - Example:
     ```
     templates/
     ├── nginx-helm/
     │   ├── Chart.yaml
     │   ├── values.yaml
     │   └── templates/
     │       ├── deployment.yaml
     │       └── service.yaml
     └── database-cr/
         ├── kustomization.yaml
         └── database-cr.yaml
     ```

2. **environments/constellations/**
   - Organizes constellations (groups of clusters) and standalone clusters.
   - Subdirectories:
     - **<constellation-name>/**: Represents a group of clusters with shared configurations.
       - `constellation.yaml`: Defines shared variables for the constellation.
       - `applications/`: Contains YAML files for applications shared across all clusters in the constellation.
       - `clusters/`: Contains subdirectories for each cluster in the constellation.
     - **standalone-clusters/**: Contains clusters not part of any constellation (e.g., central infrastructure clusters).

3. **constellations/<constellation-name>/constellation.yaml**
   - Defines shared variables for all clusters in the constellation.
   - Does not list applications (applications are defined in the `applications/` directory).
   - Example:
     ```yaml
     variables:
       env: production
       region: us-east
     ```

4. **constellations/<constellation-name>/applications/**
   - Contains one YAML file per application deployed to all clusters in the constellation.
   - Each file specifies:
     - `name`: A unique identifier for the application instance.
     - `template`: Path to the template in `templates/` (e.g., `nginx-helm`).
     - `parameters`: Key-value pairs for customizing the application, supporting variable interpolation (e.g., `{{ variables.env }}`).
   - Example (`app1.yaml`):
     ```yaml
     name: common-app
     template: common-app-template
     parameters:
       someParam: {{ variables.env }}
     ```

5. **constellations/<constellation-name>/clusters/<cluster-name>/**
   - Represents an individual Kubernetes cluster.
   - Subdirectories:
     - `cluster.yaml`: Defines the cluster’s configuration, including its constellation and cluster-specific variables.
     - `applications/`: Contains YAML files for cluster-specific applications.
   - Example `cluster.yaml`:
     ```yaml
     constellation: constellation-1
     variables:
       clusterName: cluster1
     ```
   - Example `applications/app4.yaml`:
     ```yaml
     name: app4
     template: nginx-helm
     parameters:
       replicaCount: 2
       cluster: {{ variables.clusterName }}
     ```

6. **constellations/standalone-clusters/<cluster-name>/**
   - Same structure as clusters within constellations but omits the `constellation` field in `cluster.yaml`.
   - Used for clusters not part of any group (e.g., central infrastructure).
   - Example `cluster.yaml`:
     ```yaml
     variables:
       clusterName: infra-cluster
     ```

## Key Features
- **Modular Application Definitions**: Each application is defined in its own YAML file, keeping configurations small and manageable.
- **Cluster Grouping**: Clusters are organized within constellations, with standalone clusters separated for clarity.
- **Variable Inheritance**:
  - Variables defined in `constellation.yaml` are inherited by all clusters in that constellation.
  - Cluster-specific variables in `cluster.yaml` can override or extend inherited variables.
  - Interpolation (e.g., `{{ variables.clusterName }}`) allows dynamic parameterization.
- **Reusable Templates**: Templates in `templates/` can be reused across clusters and constellations, supporting Helm, Kustomize, or manifests.
- **Scalability**: The nested structure supports a large number of clusters and applications without overwhelming directory sizes.

## Example Scenario: Database Deployment
To deploy a database operator and two instances:
1. **Template**:
   - `templates/database-operator-helm/`: Contains the Helm chart for the operator.
   - `templates/database-cr/`: Contains Kustomize configuration for database CRs.
2. **Constellation**:
   - `constellations/constellation-1/constellation.yaml`:
     ```yaml
     variables:
       env: production
     ```
   - `constellations/constellation-1/applications/db-operator.yaml`:
     ```yaml
     name: db-operator
     template: database-operator-helm
     ```
3. **Cluster**:
   - `constellations/constellation-1/clusters/cluster1/cluster.yaml`:
     ```yaml
     constellation: constellation-1
     variables:
       clusterName: cluster1
     ```
   - `constellations/constellation-1/clusters/cluster1/applications/db-instance-1.yaml`:
     ```yaml
     name: db-instance-1
     template: database-cr
     parameters:
       size: small
     ```
   - `constellations/constellation-1/clusters/cluster1/applications/db-instance-2.yaml`:
     ```yaml
     name: db-instance-2
     template: database-cr
     parameters:
       size: large
     ```

## Usage Guidelines
1. **Adding a Template**:
   - Create a new subdirectory in `templates/` (e.g., `my-app/`).
   - Add the necessary files (e.g., Helm chart, Kustomize files, or manifests).
2. **Adding a Constellation**:
   - Create a new directory under `environments/constellations/` (e.g., `constellation-2/`).
   - Add a `constellation.yaml` with shared variables.
   - Create an `applications/` directory for shared applications.
   - Create a `clusters/` directory for member clusters.
3. **Adding a Cluster**:
   - If part of a constellation, create a subdirectory under `constellations/<constellation-name>/clusters/` (e.g., `cluster2/`).
   - If standalone, create a subdirectory under `constellations/standalone-clusters/`.
   - Add a `cluster.yaml` with the `constellation` field (if applicable) and cluster-specific variables.
   - Create an `applications/` directory for cluster-specific applications.
4. **Adding an Application**:
   - Create a YAML file in the appropriate `applications/` directory (constellation or cluster level).
   - Specify `name`, `template`, and `parameters`.
5. **Using Variables**:
   - Define variables in `constellation.yaml` or `cluster.yaml`.
   - Use `{{ variables.<name> }}` in `parameters` for interpolation.

## Notes
- The repository does not include logic for reading or applying the configurations (e.g., via Argo CD). This is handled by your deployment tool.
- Ensure template paths in application YAMLs match the structure in `templates/`.
- Use descriptive names for constellations, clusters, and applications to maintain clarity.

## Contact
For questions or contributions, please reach out to the repository maintainers.
