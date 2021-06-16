## Prepare the Bastion host
Install the supporting tools on to the Bastion host required for the OpenShift install and config. 
```
sudo yum install -y git
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum install -y terraform
sudo curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
sudo yum install -y jq
sudo yum install -y wget
```
## OpenShift 4.7 User-Provided Infrastructure
Follow the Installation process from the main README.md

The terraform.tfvars has been update to close to TH variables

## Post-Install Configuration 
There are number of steps required to be done after the cluster has been sucessfully setup. 

1. Configure htpasswd file for Auth
    TH needs to create the App Registration and assign permissions.

    Can be done using Azure AZ CLI or GUI:
    ```
    az ad app create --display-name thupi-ad --homepage https://console-openshift-console.apps.thupi.im.ocp --reply-urls https://oauth-openshift.apps.thupi.im.ocp/oauth2callback/azureAD --identifier-uris https://console-openshift-console.apps.thupi.im.ocp --password '#PASSWORD#'
    ```
    - Add AZ Graph API permission and Grant Admin Consent.


2. Configure Azure AD for Auth 
    Can be done using oc or console:
    - Goto Cluster settings and oAuth, add a new oAuth type and update with - azure-ad-auth.yaml. 
    - oc CLI method
    ```
    oc create secret generic openid-client-secret-azuread -n openshift-config --from-literal=clientSecret=#PASSWORD#
    ```
3. Configure the logout URL in OpenShift to properly logout the SSO:
    ```
    oc edit console.config.openshift.io cluster
    spec:
    authentication:
    logoutRedirect: https://login.microsoftonline.com/common/oauth2/v2.0/logout?post_logout_redirect_uri=https://console-openshift-console.apps.thupi.im.ocp
    ```
    [Logout-Ref-Doc] https://docs.openshift.com/container-platform/4.7/web_console/configuring-web-console.html

4. Configure NFS PV/PVC
    - Provision the NFS server 
        TH will provision the NFS server and provide IP and MountPoints. 
    
    - Create mount point and export
        TH will create mount point for /registry and /acd and exports.
        
        Create PV with required size and create PVC (it will bound automatically when the size is same)
        ```
        oc create -f manifests/nfs-file-store-pv.yaml
        oc create -f manifests/nfs-file-store-pvc.yaml
        ```
5. Update OpenShift Registry to use NFS
    - OpenShift registry is removed during install, and needs to be set back to PV for image storage
    ```    
    oc edit configs.imageregistry.operator.openshift.io
    ```
    - Change the keys as below:
    ```
    managementState: Managed

    storage:
    pvc:
        claim: # leave the claim blank
    ```
    - Create the volume
    ```
    oc create -f manifests/registry-pv.yaml
    ```

## ACE SSO Configuration 
ACE require User authentication, this can be setup with Azure AD.
   
1. In the already create Azure AD App Registration add the ACE redirect URI - `https://#ACD SERVER#/services/redirect_uri`.
2. Install OpenIDC Module on the HTTPD server
    ```
    sudo yum install mod_auth_openidc
    sudo dnf module enable mod_auth_openidc
    ```
  Edit the `/etc/httpd/conf.d/auth_openidc.conf` with the values from Azure AD App:

| Variable                          | Value                                                | 
| --------------------------------  | -----------------------------------------------------|    
| OIDCRedirectURI                   | https://172.16.244.208/services/redirect_uri            | 
| OIDCCryptoPassphrase              | Wats0n          | 
| OIDCProviderMetadataURL           | https://login.microsoftonline.com/#TENANT_ID#/v2.0/.well-known/openid-configuration |
| OIDCClientID                      | #APP REG CLIENT ID#                                       | 
| OIDCClientSecret                  | #USE FOR CREATING APP REG#                                | 

 At the top of the same file add:
 ```
   # echo back the claim headers to the client so it knows the user id and email  
   Header echo ^OIDC_CLAIM_ 
    
   <Location /services>
        AuthType openid-connect
        Require valid-user
    </Location>
```