# Why do we need custom signing keys?

Custom Signing keys becomes an important step when we look at configuring claims mapping policies for applications. Application developers can use optional claims in their Azure AD applications to specify which claims they want in tokens sent to their application.

You can use optional claims to:

  1. Select additional claims to include in tokens for your application.
  2. Change the behavior of certain claims that the Microsoft identity platform returns in tokens.
  3. Add and access custom claims for your application.

After creating custom claims mapping policies for applications, during user sign in, you may run into the following error in your application:

    "AADSTS50146: This application is required to be configured with an application-specific signing key. It is either not configured with one, or the key has expired or is not yet valid."
  
To overcome this error, the general guidance is to modify the app manifest file to set "acceptMappedClaims" attribute to TRUE.

However the documentation also shares another warning as below -

    Warning: Do not set acceptMappedClaims property to true for multi-tenant apps, which can allow malicious actors to create claims-mapping policies for your app.

Hence in case of multi-tenant apps, the guideline is to use a custom signing key instead of modifying the manifest file.

The following portion of this doc provides detailed guidelines on how to create custom signing keys and then also demonstrates how you could create claim mapping policies for an application and try to fetch an access token without hitting the above specfied error.


## The first step here is to have a certificate with public and private key exported 

The following code can be used to create a self-signed certificate which is the first step:

    Param(
    [Parameter(Mandatory=$true)]
    [string]$fqdn,
    [Parameter(Mandatory=$true)]
    [string]$pwd,
    [Parameter(Mandatory=$true)]
    [string]$location
    ) 

    if (!$PSBoundParameters.ContainsKey('location'))
      {
        $location = "."
      } 

    $cert = New-SelfSignedCertificate -certstorelocation cert:\currentuser\my -DnsName $fqdn
    $pwdSecure = ConvertTo-SecureString -String $pwd -Force -AsPlainText
    $path = 'cert:\currentuser\my\' + $cert.Thumbprint
    $cerFile = $location + "\\" + $fqdn + ".cer"
    $pfxFile = $location + "\\" + $fqdn + ".pfx" 

    Export-PfxCertificate -cert $path -FilePath $pfxFile -Password $pwdSecure
    Export-Certificate -cert $path -FilePath $cerFile
    
Once this has been done, we need to extract the base64 encoded values of the public-private keys, which is then used to create the certificate thumbprint. This inturn is used to build our payload which would go into the keyCredential attribute of the application/service principal to signify the application's authentication credentials.


### 1. Create JSON payload from certificate:

  Pre-reqs: 
  Certificate exported with and without private key (.pfx and .cer) 
  PFX requires PKCS12 format 

  Open PowerShell and execute the below script -

      $pfxFile = "C:\folder\certificate.pfx" 
      #Pfx file location 

      $cerFile = "C:\folder\certificate.cer" 
      #Cer file location 

      $payload = "C:\folder\payload.txt" 
      #File which payload will be exported to (should not already exist, will be created) 

      $pwd="P@SSw0rd" 
      #Password for pfx file 

      $cert=Get-PfxCertificate -FilePath $pfxFile 
      #Gets certificate details 

      #Create base64 of PRIVATE KEY 
      $pfx_cert = get-content $pfxFile -Encoding Byte 
      $base64pfx = [System.Convert]::ToBase64String($pfx_cert) 

      #Create base64 of PUBLIC KEY 
      $cer_cert = get-content $cerFile -Encoding Byte 
      $base64cer = [System.Convert]::ToBase64String($cer_cert)

      Build JSON payload # getting id for the keyCredential object 
      $guid1 = New-Guid 
      $guid2 = New-Guid 

      Get the custom key identifier from the certificate thumbprint: 
      $hasher = [System.Security.Cryptography.HashAlgorithm]::Create('sha256') 
      $hash = $hasher.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($cert.Thumbprint)) 
      $customKeyIdentifier = [System.Convert]::ToBase64String($hash) 

      Get end date and start date for our keycredentials 
      $endDateTime = ($cert.NotAfter).ToUniversalTime().ToString( "yyyy-MM-ddTHH:mm:ssZ" ) 
      $startDateTime = ($cert.NotBefore).ToUniversalTime().ToString( "yyyy-MM-ddTHH:mm:ssZ" ) 

  Building our json payload 

      $object = [ordered]@{ 
          keyCredentials = @( [ordered]@{ 
                customKeyIdentifier = $customKeyIdentifier 
                endDateTime = $endDateTime 
                keyId = $guid1 
                startDateTime = $startDateTime 
                type = "X509CertAndPassword" 
                usage = "Sign" 
                key = $base64pfx 
                displayName = "CN=certificate.local" }

                , [ordered]@{ 
                  customKeyIdentifier = $customKeyIdentifier 
                  endDateTime = $endDateTime 
                  keyId = $guid2 
                  startDateTime = $startDateTime 
                  type = "AsymmetricX509Cert" 
                  usage = "Verify" 
                  key = $base64cer 
                  displayName = "certificate.local" 
                } ) 

                  passwordCredentials = @( [ordered]@{ 
                    customKeyIdentifier = $customKeyIdentifier 
                    keyId = $guid1 
                    endDateTime = $endDateTime 
                    startDateTime = $startDateTime 
                    secretText = $pwd } ) } 

      $json = $object | ConvertTo-Json -Depth 99 

Export JSON payload to a text file for use with Microsoft Graph
    
    $json | Out-File $payload
    

### 2. Upload custom signing key via Microsoft Graph:

Pre reqs: ServicePrincipal ObjectID

Connect to Graph Explorer as user with appropriate permissions on target ServicePrincipal.

Confirm SPOID returns correct by executing- 
GET https://graph.microsoft.com/v1.0/serviceprincipals/SPOID 

Set the following header:
Content-type: application/json 


Paste the body in the payload file into the request body, change the GET action to PATCH and then run the query.
The response should be: No content - 204 


This means we have successfully linked the serviceprincipal object's credentials to the certificate used.


#### **Note:** As part of the OIDC protocol, applications utilize the OpenID Provide Configuration Document at a publicly accessible endpoint containing the provider's OIDC endpoint, supported claims, and other metadata. Client apps use this metadata to discover the URLs to use for authentication and the authentication service's public signing keys. Every app registration in Azure AD is provided a publicly accessible endpoint that serves its OpenID configuration document. To determine the URI of the configuration document's endpoint for your app, append the well-known OpenID configuration path to your app registration's authority URL.


  - Well-known configuration document path: /.well-known/openid-configuration
  - Authority URL: https://login.microsoftonline.com/{tenant}/v2.0
  

The value of {tenant} varies based on the application's sign-in audience. In case of multitenant apps, where users with both personal MS account and work or school account can sign in, the value is always common. Hence in our case, the app fetches the metadata from:

  [https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration](https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration)

Now when we updated the service principal's keycredential attribute, the custom signing keys are appended to this URL:

https://login.microsoftonline.com/{tenantID}/discovery/v2.0/keys?appid={appID} 





### 3. Now let us create a Claims Mapping Policy and assign it to ServicePrincipal:

Pre reqs: ServicePrincipal ObjectID

  - Open PowerShell 
  - Connect to tenant as user with appropriate permissions. Run:
        Connect-AzureAD 

**Check if the SP currently has policy applied**

      Get-AzureADServicePrincipalPolicy -Id SPOID 

**Create a new Claims Mapping Policy and set to a variable**

      $pol = New-AzureADPolicy -Definition @('{"ClaimsMappingPolicy":{"Version":1,"IncludeBasicClaimSet":"true"}}') -DisplayName "SigningKeyPolicy" -Type "ClaimsMappingPolicy" 

**Alternatively, use existing Claims Mapping Policy**

      Get-AzureADPolicy 
      List all policies, locate existing policy Id and use with: 
      $pol = Get-AzureADPolicy -Id xxx 

**Assign the policy to the SP**

      Add-AzureADServicePrincipalPolicy -Id SPOID -RefObjectId $pol.Id 

**Confirm the policy applied correctly**

      Get-AzureADServicePrincipalPolicy -Id SPOID 
      


### 4. Now request a token for the application and verify the signature:


a. Running the application in question will ask for the user to enter his/her credentials post which they should get a token successfully with the custom claims embedded. They would no longer see the error about application needing to be configured with a custom signing keys.

In case you have implemented token validation logic, the user would also see another error in the application like:

    An unhandled exception occurred while processing the request.
		SecurityTokenSignatureKeyNotFoundException: IDX10501: Signature validation failed. Unable to match key:
		kid: '[PII is hidden. For more details, see https://aka.ms/IdentityModel/PII.]'.
		Exceptions caught:
		'[PII is hidden. For more details, see https://aka.ms/IdentityModel/PII.]'.
		token: '[PII is hidden. For more details, see https://aka.ms/IdentityModel/PII.]'.
			System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler.ValidateSignature(string token, TokenValidationParameters validationParameters)
		Exception: An error was encountered while handling the remote login. 
    Microsoft.AspNetCore.Authentication.RemoteAuthenticationHandler<TOptions>.HandleRequestAsync()![image](https://github.com/Mansv/AzureAD_Docs/assets/70005633/fcedb8d0-8782-4033-ae3e-88b6ff8630ed)
   
This is because the application has been signed with a custom key but does not find it in the known OIDC metadata endpoint.

To overcome the signature validation error, you could modify the app config file to include an attribute that explicitly asks the application to fetch metadata from the endpoint where we have configured the custom signing keys. The following attribute needs to be included -

    "AzureAd": {
	        "Enabled": true,
	        "AuthenticationType": "AzureAD",
	        "AuthenticationCaption": "Azure Active Directory",
	        "ApplicationId": "enter_application_ID",
	        "TenantId": "enter_tenant_ID",
	        "AzureAdInstance": "https://login.microsoftonline.com/",
	        *"MetadataAddress": "https://login.microsoftonline.com/{tenantID}/v2.0/.well-known/openid-configuration?appid={appID}",*
	        "UsePreferredUsername": false
    },

After updating the config file, the signature validation error should be overcome.



  b. Using Postman tool:
  Pre reqs: App ID, Client Secret

    1. Open Postman 
    2. Use the Client Credential with shared secret flow to request a token with the following key:value pairs 

          - Grant_type: client_credentials 
          - Client_id: AppID 
          - Scope: AppID/.default 
          - Client_secret: ClientSecret 

**NOTE:** Scope cannot be MS Graph (for example user.read) as these scopes will ALWAYS be signed by AzureAD. 

Copy the received token and paste into https://jwt.ms to decode.

Check the KID against the URL.

The token would be issued with the custom signing key (kid value) and would include the custom claim from the claim mapping policy. However token validation step would still fail unless the app has been configured to look at the appID-specific endpoint for the metadata.


#### References:
1. OIDC protocol(https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-protocols-oidc)
2. Validating id token(https://learn.microsoft.com/en-us/azure/active-directory/develop/id-tokens#validate-tokens)





