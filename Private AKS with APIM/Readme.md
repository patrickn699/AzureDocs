## Documentation for creating private aks having interal/private NGINX ingress along with self-signed certificates and exposing the ingress using APIM.


### Installing NGINX ingress and maing it internal.

1. Assuming that private aks is already available and helm installed, let's start by intalling NGINX ingress controller on the same.

```kubectl
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-basic --create-namespace --namespace default
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
--set controller.nodeSelector."kubernetes\.io/os"=linux
--set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux
--set defaultBackend.nodeSelector."kubernetes\.io/os"=linux --set controller.service.loadBalancerIP="" 
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true # this assings a private ip from the vnet.


kubectl get deployments && kubectl get svc

```
you should get the following output for deployments and service.

![deployments](./images/nginx1.png)

![deployments](./images/nginx2.png)

2. Install basic container image on AKS
   * Head to yml_files folder and apply the demo.yml file which will pull and deploy basic image and expose it as ClusterIP.

    ![img_deployments](./images/aksimg1.png)

    ![img_deployments](./images/aksimg2.png)


3. Create Ingress route
    * In order to access these deployments through ingress we need to create routes so that the incomming traffic can reach the demo service.
    * Head to yml_files folder and apply ingress.yml file & run kubectl describe ingress.
  
    * ![ing_deployments](./images/ingress1.png)
    * ![ing_deployments](./images/ingress2.png)
    * If everything is correctly installed you should be able to access it using the private ip of the ingress.
  
    * ![op_deployments](./images/op1.png)


4. Install certmanager in the cluster
   * Now we are good to go for installing cert manager on the aks, run the below command.
   * ```kubectl
      kubectl label namespace ingress-basic cert-manager.io/disable-validation=true
      # Add the Jetstack Helm repository
      helm repo add jetstack https://charts.jetstack.io
      # Update your local Helm chart repository cache
      helm repo update
      # Install CRDs with kubectl
      kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml
      # Install the cert-manager Helm chart
      helm install cert-manager jetstack/cert-manager --namespace ingress-basic --version v1.7.1
      # List the installed charts
      helm list
      # verify if certmanager is installed
       kubectl get deployments && kubectl get svc
     ```

    * ![helm](./images/helm1.png)
    * ![helm](./images/certm1.png)


5. Create ClusterIssure and self-signed Certificate
     * Now its time to crate the certificate, go the yml_files folder and apply selfi.yml followed by cert.yml.
     * you should see the below output once created.
       
     * ![issuer](./images/issuer1.png)
     * ![cert](./images/cert1.png)

6. Create A record
     * Create an A record in private dns zone of aks with the private ip of ingress controller
     * for ex. ingresspvtaks.9d4226bf-19be-4b4f-8bfc-10cb64afde86.privatelink.eastus.azmk8s.io.

7. Redeploy ingress
     * Now we need to redeploy the ingress adding the following specs to each route.
     * This will start referring to the secret where the certs are installed.
     * ```yaml
       tls:
        - hosts:
          - ingresspvtaks.9d4226bf-19be-4b4f-8bfc-10cb64afde86.privatelink.eastus.azmk8s.io
          secretName: tlss-secret-aks
       ```
8. Download and install the ingress certificate
     * After redeploying ingress, inorder for browsers to access it service via https we need to download and install the certificates on you local machine so we will be having the private key to check the signature of the certificate.
     *  To download the certificate run the below commands
     *  
       ```kubectl
               kubectl get secret -o jsonpath="{.data['ca\.crt']}"
               [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("repace this part with base 64 output of  the above command")) | Set-Content -Path ca.crt
               kubectl get secret -o jsonpath="{.data['tls\.key']}"
               [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("repace this part with base 64 output of    the above command")) | Set-Content -Path tls.key
       ```
       * Once we have saved it locally in windows os it needs to be installed.
         *  ![dwcert](./images/dicert1.png)
         *  ![dwcert](./images/tkey.png)
       *  Click on the downloaded cert to install, then select install, select local machine, select browse and click on trusted root certificate authority.
         * ![dwcert](./images/dicert2.png)
         * ![dwcert](./images/dicert3.png)
         * ![dwcert](./images/dicert4.png)
       * Next up, we should be able to see the certificate in the browser like shown below.
         * ![vcert](./images/vcert.png)
       * If you are able to see the certificate this means https is enabled to the ingress but you will see it in the red as "not secured" which means even tough its https but the certificate signature is not signed/validate by a certificate authority.

9. Adding certificates to APIM
    * Till now we are able to call the ingress with https through the browser of the vm in the sam vnet. But we also need to invoke it through APIM.
    * And to invoke it thorugh APIM we also the certificate to ensure authenticity of the APIM backend and the ingress url as "https" one.
    * Before addding certificates to the APIM we need the certificates to be in the format of .pfx. To do the same run the below commands. But make sure you have OpenSSL installed on your local machine where the certificates are saved.
    * ```
        openssl x509 -in ca.crt -out ../ca.pem
        openssl rsa -in tls.key -out ../tls.pem
        openssl pkcs12 -export -out cet.pfx -inkey ../tls.pem -in ../ca.pem -> enter a password and keep it safe
      
      ```
    * Save the .pfx key locally and head to apim assuming you already have configured the http api for aks service.
    * Go to "custom domains" blade in apim and select the existing domain, tick the box "Negotiate Client Certificate" and click update.
      *  ![apic](./images/apic1.png)
    * Then its time to upldad the certificate, head to "certificates" blade and click on add certificate option, select custom certificate and upload the certificate .pfx file from local machinea and put the password. Once done click on save.
      *  ![apic](./images/apic2.png)
    * As we are using self signed certificates, the apim will throw 500 internal error as its not verified by authority. To bypass this go to "backend" blade select the "properties" option add 'https' to the backend url and un-tick "validate certificate chain" & "validate certificate name".
      *  ![apic](./images/apic3.png)
    * Lastly head to "api" blade and edit the backend of the api, replace the url will https and add "Gateway Credentials" as "client certificate" from the dropdown select the uploaded certificate in the previous steps and click on save. Also go to "settings" in "api" blade and replace the url with 'https' click on save.
      *  ![apic](./images/apic4.png)
      *  ![apic](./images/apic5.png)
    * Now we should be able to call the aks service with https
        *  ![apic](./images/apio1.png)
        *  ![apic](./images/apio2.png)
          
9.1  Disable public network access for AKS
     * as our aks is private but there might be public network access enabled for it, to verify use the below command.

     ``` az
          az resource show --ids "aks id"
          az resource update --ids "aks id" --set properties.publicNetworkAccess=disabled
     ```
     
    

  




