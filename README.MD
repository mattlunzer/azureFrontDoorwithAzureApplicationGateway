## Azure Front Door WITH Azure Application Gateway Architecture and Configuration Guide

### Purpose

The purpose of this document is to offer a guide for the configuration of two key Azure load balancing services, Azure Front Door AND Azure Application Gateway. Throughout Microsoft documentation you will find design patterns that promote this capability and the benefits it offers.

For example, as of the time of this writing, the URL below illustrates Azure Front Door routing http traffic to Azure Application Gateway which routes http traffic to backend web services. 

- https://docs.microsoft.com/en-us/azure/frontdoor/front-door-lb-with-azure-app-delivery-suite#building-with-azures-application-delivery-suite

However, there is limited how-to documentation to impleemnt this configuration end to end (with end to end SSL). This guide is intended to account for that.

## Key Components

### Azure Front Door

This is one of my favorite Azure Services. Designed by Microsoft for some of the largest web applications on the planet, Front Door offers unparalled performance and security gains and is now available in Azure to provide the same benefits to customer applications. I'm not going to do it justice, and your not here for that. See more at the URL below.

- https://azure.microsoft.com/en-us/services/frontdoor/

### Azure Application Gateway

Whereas Azure Front Door is global, Azure Application Gateway is regional. Azure Application Gateway is also a layer 7 application delivery controller and has some added features such as the ability to perform connection draining, act as an ingress controller for Azure Kubernetes Service and can publically expose a private backend (whereas Front Door requires a public endpoint). See more at the URL below.

- https://azure.microsoft.com/en-us/services/application-gateway/

### Your Web Application

The intent of this architecture is to expose an external facing web application. Therefore, your web application, whatever it is comprised of, is the third layer of this infrastructure pattern. Possibilities include;

- App Service
- Containerized services
- Virtual Machines

Like many services in Azure, Azure Front Door and Azure Application Gateway are hybrid and support web applications that run in and outside of Azure, including in customer data centers or 3rd party cloud platforms such as aws. 

## The Design Architecture

The assumption in this design is as follows.

- (Internet --> Azure Front Door --> Azure Application Gateway --> Web App)

### Architecture Diagram

![GitHub Logo](/images/afd_highlevel.jpg)

Now let's get down to the configuration.

Tip - Deploy the Web Application, Azure Application Gateway, and Azure Front Door, in that order. At each step, apply your custom domain (e.g. customer.com), your SSL certificate (*.customer.com), point your public dns record to the service and test. Not very DevOps friendly, but helpful if you are verifying your architecture configuration and or learning. 

### Azure Web App Configuration

1. Configure the web application in accordance to your standards.

2. Configure a custom domain name. In the case of Azure App Service, here is a good good walkthrough. https://docs.microsoft.com/en-us/Azure/app-service/app-service-web-tutorial-custom-domain

    Tip - if your web url is already mapped to Front Door, you can use awverify to add a host header to App Services
<https://docs.microsoft.com/en-us/Azure/app-service/web-sites-traffic-manager-custom-domain-name>

3. For end-to-end SSL, upload your certificate. Here is a good walkthrough https://docs.microsoft.com/en-us/Azure/app-service/configure-ssl-certificate

    Tip - if you want to offload SSL, you can do so on Front Door or Application Gateway. In that case, no certificate is required as traffic will flow to your application unencrypted over http. While this will alleviate the cryptographic overhead on your web farm (e.g. CPU cycles) you must accept the risk associated with traffic from your ADC being unencrypted.

### Azure Application Gateway Configuration

1. Deploy Azure Application Gateway (v2) in accordance to your standards. Note that the Application Gateway MUST have a public IP address (or Front Door will not be able to reach it). Note the public IP is static (we will use it for DNS A record in step 3).

2. During provisioning upload or use your certificate stored in Azure Key Vault (Managed Identity Required) for your https listener. You will need the certificate to support SSL traffic coming from Azure Front Door. Unless you are using named certificates, upload a multi-domain (e.g. SAN) or wildcard certificate. This will save you a bit of time for the next step. 

#### Important - Configure a listener for the Front Door Health Probe

3. Once Application Gateway is successfully deployed, configure a listner for the Azure Front Door health probe. Map a custom domain name via a unique DNS A record in your public DNS (e.g appgw.customer.com) and ensure it associates with the certificate you already uploaded during provisioning, or upload a new certificate. Failure to configure a lister with a properly configured custom domain name and ssl certificate will cause the Azure Front Door health probe to fail which renders the backend unavailable.

