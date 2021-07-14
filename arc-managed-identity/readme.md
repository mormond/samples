
# Use Azure Arc and Managed Identity to access Azure resources from an on-prem Linux VM

1. We will simulate an on-prem VM in Azure
    1. Create an Ubuntu 18.04 LTS VM in Azure
    1. Change the hostname to something memorable & unique (we will need this later)
    1. Install jq
        ```
        sudo apt install jq
        ```
    3. Follow the instructions at [Evaluate Arc enabled servers on an Azure virtual machine
](https://docs.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine) to create an Azure VM that can be managed by Azure Arc
    1. Confirm you can see the "on-prem" VM in the Servers - Azure Arc blade in the Azure Portal
1. Create the protected resource (Key Vault) we want to be able to access
    1. Create a Key Vault instance and add a secret called "Secret1" - this will be the protected resource we access
    1. Add an access policy to the Key Vault adding at least Get permissions for Secrets
    1. For the Principal, search for the memorable hostname of the Arc enabled VM you created in the previous step
    1. This grants the Managed Identity of the VM access to read the Key Vault secrets
1. Get an Auth Token to access Key Vault from the VM
    1. We use the Azure Instance Metadata Service (IMDS) on the VM to get hold of the identity and auth token
    1. The IMDS endpoint is `http://locahost:40342`
    1. The identity endpoint is `http://localhost:40342/metadata/identity/oauth2/token`
    1. It should be possible to use ENV variables IMDS_ENDPOINT and IDENTITY_ENDPOINT but I found they weren't populated during my testing
    1. Code running on the server can use the endpoint to request an auth token to access Azure resources
    1. In order to acquire a token, the client making the request must provide a secret (challenge token) to prove they are authorised
    1. To be able to access the secret, the client must be a member of the `himds` group
    1. SSH to the "on-prem" VM
    1. Add the current user to the `himds` (eg for the default username of `azureuser`)
        ```
        sudo usermod -a -G himds azureuser
        ```
    1. Logout and back in again for the change to take effect
    1. Run the following commands 
        ```
        ChallengeTokenPath=$(curl -s -D - -H Metadata:true "http://127.0.0.1:40342/metadata/identity/oauth2/token?api-version=2019-11-01&resource=https%3A%2F%2Fmanagement.azure.com" | grep Www-Authenticate | cut -d "=" -f 2 | tr -d "[:cntrl:]")
        echo $ChallengeTokenPath
        ```
    1. You should have a populated path - this path is secured. Access to the contents is required to be able to request an OAuth token
    1. Run the following commands
        ```
        ChallengeToken=$(cat $ChallengeTokenPath)
        echo $ChallengeToken
        ```
    1. You should have a token. We can use this token to authorise a request for an OAuth token for Key Vault
        ```
        curl -s -H Metadata:true -H "Authorization: Basic $ChallengeToken" "http://127.0.0.1:40342/metadata/identity/oauth2/token?api-version=2019-11-01&resource=https%3A%2F%2Fvault.azure.net" | jq '.access_token'
        ```
    1. This will return just the bearer token required for the next step
1. Make a request for the Key Vault secret
    1. Grab the URL of the Key Vault you created in an earlier step - something like `https://mykv.vault.azure.net/` 
    1. Excute the following command
        ```
        curl 'https://<YOUR-KEY-VAULT-URL>/secrets/<secret-name>?api-version=2016-10-01' -H "Authorization: Bearer <ACCESS TOKEN>"
        ```
    1. Where `<YOUR-KEY-VAULT-URL>` is the Key Vault URL and `<ACCESS TOKEN>` is the bearer token from the previous step
    2. You should get a response similar to
        ```
        {
            "value":"123",
            "id":"https://mykv.vault.azure.net/secrets/Secret1/25332ae471a14492bdd87f5f64ea758a","attributes":{"enabled":true,"created":1626261766,"updated":1626261766,"recoveryLevel":"Recoverable+Purgeable"}
        }
        ```

## References

* [Evaluate Arc enabled servers on an Azure virtual machine
](https://docs.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine)
* [Using a managed identity with Arc enabled servers](https://docs.microsoft.com/en-us/azure/azure-arc/servers/security-overview#using-a-managed-identity-with-arc-enabled-servers)
* [Authenticate against Azure resources with Arc enabled servers
](https://docs.microsoft.com/en-us/azure/azure-arc/servers/managed-identity-authentication)
* [Tutorial: Use a Linux VM system-assigned managed identity to access Azure Key Vault
](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/tutorial-linux-vm-access-nonaad)
