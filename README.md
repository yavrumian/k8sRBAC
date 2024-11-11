# Kubernetes RBAC & AWS RDS Integration

This repository provides a sample Kubernetes configuration to demonstrate Role-Based Access Control (RBAC) and secure integration with AWS RDS for a multi-environment setup. The solution covers two environments (`staging` and `production`) and two roles (`developer` and `admin`), each with specific access permissions. Additionally, two options are provided for securely accessing AWS RDS from Kubernetes.

## Project Structure

- `main-config.yaml`: Contains the core configuration for namespaces, roles, role bindings, and service accounts.
- `rds-option-1.yaml`: Implements Option 1 for accessing RDS using AWS Secrets Manager.
- `rds-option-2.yaml`: Implements Option 2 for accessing RDS using a Service Account (SA) with an IAM role.

## RBAC Permissions

The following RBAC permissions are configured in the `main-config.yaml` file:

1. **Namespaces**: Defines separate namespaces for `staging` and `production`.
2. **Roles**:
   - **Developer** in `staging` namespace: Full access to all resources.
   - **Admin** in both `staging` and `production` namespaces: Full access to all resources.
3. **Service Accounts**:
   - `developer-sa` (for developers) in `staging` namespace.
   - `admin-sa` (for admins) in both `staging` and `production` namespaces.
4. **RoleBindings**:
   - `developer-sa` is bound to the Developer role in the `staging` namespace.
   - `admin-sa` is bound to the Admin role in both `staging` and `production`.

## AWS RDS Access Options

### Option 1: Using AWS Secrets Manager with Secrets Store CSI Driver
This approach retrieves RDS credentials from AWS Secrets Manager and mounts them as volumes in the pods.

- **Prerequisites**: This setup assumes that the [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/) is already installed in the cluster along with the AWS provider plugin.
- **Configuration**: The file `rds-option-1.yaml` sets up a `SecretProviderClass` for each environment (staging and production) to retrieve RDS credentials from AWS Secrets Manager.
- **Usage**: Pods use the mounted secrets to access the RDS database securely.

**Pros**:
- **Centralized Secret Management**: Credentials are securely managed in AWS Secrets Manager, allowing centralized control, rotation, and auditing of secrets.
- **Automatic Rotation**: AWS Secrets Manager can automatically rotate RDS credentials, which the Secrets Store CSI Driver can sync to Kubernetes without requiring pod restarts.
- **Secure Storage**: Secrets are stored securely in AWS Secrets Manager, reducing the risk of exposure in Kubernetes.

**Cons**:
- **Dependency on CSI Driver**: This setup requires additional components (Secrets Store CSI Driver and AWS provider plugin), which add complexity and require ongoing maintenance.
- **Potential Latency**: Accessing secrets via the CSI driver can introduce slight latency when retrieving credentials from AWS Secrets Manager, especially for high-frequency workloads.

### Option 2: Using Service Account with IAM Role
In this approach, a Service Account in Kubernetes assumes an IAM role with permissions to access RDS directly.

- **Prerequisites**: This setup assumes the availability of an IAM role with RDS access permissions and that IAM role-ARN annotations for Service Accounts are configured in AWS EKS (if using EKS).
- **Limitations**: Not all AWS RDS engines support IAM authentication. Currently, IAM-based authentication is supported for Amazon Aurora MySQL, Amazon RDS MySQL, and Amazon RDS PostgreSQL. Other databases, like Amazon RDS for Oracle or SQL Server, do not support IAM database authentication.
- **Configuration**: The file `rds-option-2.yaml` sets up a Service Account with the IAM role annotation, allowing it to directly access the RDS instance using IAM authentication if supported by the RDS engine.

**Pros**:
- **No Credentials Stored**: This method eliminates the need to store RDS credentials in Kubernetes or AWS Secrets Manager, reducing the risk of credential exposure.
- **Direct IAM Authentication**: If the RDS engine supports IAM authentication, the application can directly connect to RDS using IAM-based authentication, simplifying access control.

**Cons**:
- **Limited RDS Engine Support**: IAM authentication is only supported for Amazon RDS MySQL, Amazon Aurora MySQL, and Amazon RDS PostgreSQL. Other RDS engines (e.g., Oracle, SQL Server) do not support this approach.
- **AWS-Specific Solution**: This method relies on AWS IAM, so it’s not portable to other cloud providers or on-premises Kubernetes clusters.
- **Potential Complexity with IAM Policies**: Managing granular IAM policies to limit access to only necessary databases can be complex, especially in multi-environment setups.

## Additional Notes

- **RBAC Best Practices**: Each role is scoped to the specific permissions required for the given namespace. The use of RoleBindings ensures that each Service Account is limited to only the necessary resources, following the principle of least privilege.
- **Secret Management Best Practices**: Secrets are managed using Kubernetes Secrets for `Option 1`, stored in AWS Secrets Manager, which provides rotation, auditing, and secure access controls.
- **Compatibility**: The configurations provided are compatible with most Kubernetes distributions, but certain features (like IAM roles for Service Accounts) are specific to AWS EKS.