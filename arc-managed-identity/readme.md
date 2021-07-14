
Use Azure Arc with an on-prem Linux VM to use Managed Identity to access Azure resources

1. We will simulate an on-prem VM in Azure
    1. Create an Ubuntu 18.04 LTS VM in Azure
    1. Change the hostname to something memorable & unique (we will need this later)
    1. Follow the instructions at [Evaluate Arc enabled servers on an Azure virtual machine
](https://docs.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine) to create an Azure VM that can be managed by Azure Arc
    1. Confirm you can see the "on-prem" VM in the Servers - Azure Arc blade in the Azure Portal
1. Create the protected resource (Key Vault) we want to be able to access
    1. Create a Key Vault instance and add a secret called "Secret1" - this will be the protected resource we access
    1. Add an access policy to the Key Vault adding at least Get permissions for Secrets
    2. For the Principal, search for the memorable hostname of the Arc enabled VM you created in the previous step
    3. This grants the Managed Identity of the VM access to read the Key Vault secrets
1. dsds
    3. We use the Azure Instance Metadata Service (IMDS) on the VM to get hold of the identity and auth token
    4. The IMDS endpoint is `http://locahost:40342`
    5. The identity endpoint is `http://localhost:40342/metadata/identity/oauth2/token`
    6. It should be possible to use ENV variables IMDS_ENDPOINT and IDENTITY_ENDPOINT but I found they weren't populated during my testing
    7. Code running on the server can use the endpoint to request an auth token to access Azure resources
    8. In order to acquire a token, the client making the request must provide a secret (challenge token) to prove they are authorised
    9. SSH to the "on-prem" VM
    10. Run the following commands 
    ```
    ChallengeTokenPath=$(curl -s -D - -H Metadata:true "http://127.0.0.1:40342/metadata/identity/oauth2/token?api-version=2019-11-01&resource=https%3A%2F%2Fmanagement.azure.com" | grep Www-Authenticate | cut -d "=" -f 2 | tr -d "[:cntrl:]")
    echo $ChallengeTokenPath
    ```
    1. You should have a populated path
    1. Run the following commands
    ```
    ChallengeToken=$(cat $ChallengeTokenPath)
    echo $ChallengeToken
    ```
    1. You need root permissions so if this doesn't work, try
    ```
    ChallengeToken=$(sudo cat $ChallengeTokenPath)
    echo $ChallengeToken    
    ```
    1. You should have a token. We now need an OAuth token to authorise us for Key Vault
    ```
    curl -s -H Metadata:true -H "Authorization: Basic $ChallengeToken" "http://127.0.0.1:40342/metadata/identity/oauth2/token?api-version=2019-11-01&resource=https%3A%2F%2Fvault.azure.net"
    ```
