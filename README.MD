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

- (Internet --> Azure Front Door --> Azure Application Gateway --> (Azure) Web App)

### Architecture Diagram

![GitHub Logo](/images/afd_highlevel.jpg)

Now let's get down to the configuration.

Tip - Deploy the Web Application, Azure Application Gateway, and Azure Front Door, in that order. At each step, apply your custom domain (e.g. contoso.com), your SSL certificate (*.contoso.com), point your public dns record to the service and test. Not very DevOps friendly, but helpful if you are verifying your architecture configuration and or learning. 

### Azure Web App Configuration

1. Configure the web application in accordance to your standards.

2. Configure a custom domain name. In the case of Azure App Service, here is a good good walkthrough. https://docs.microsoft.com/en-us/Azure/app-service/app-service-web-tutorial-custom-domain

    Tip - if your web url is already mapped to Front Door or another service, you can use awverify to add a host header to App Services
<https://docs.microsoft.com/en-us/Azure/app-service/web-sites-traffic-manager-custom-domain-name>

3. For end-to-end SSL, upload your certificate. For App Service, here is a good walkthrough https://docs.microsoft.com/en-us/Azure/app-service/configure-ssl-certificate

    Tip - if you want to offload SSL, you can do so on Front Door or Application Gateway. In that case, no certificate is required for your web application as traffic will flow  unencrypted over http. While this will alleviate the cryptographic overhead on your web farm (e.g. CPU cycles) you must accept the risk associated with traffic from Application Gateway being unencrypted.
    
    *Given decision point above know that (natively) any traffic between Azure Services does NOT traverse the public Internet*

### Azure Application Gateway Configuration

1. Deploy Azure Application Gateway (v2) in accordance to your standards. Key configuration points.
    
    A. Deploy Azure Application Gateway v2. Many benefits over v1. Include the Web Application Firewall if you wish (suggested).

    B. Provision a public instance. A public instance includes a static public IP address. Recall that Azure Front Door requires a publically accessible backend. Also, the IP address is used needed for the DNS configuration.

    C. Upload your certificate or refer to Azure Key Vault (requires Managed Identity). If possible, use a multi-site (e.g. SAN) or wildcard certificate (*.contoso.com) otherwise you will need a second certificate for the health probe listiner.
    
    D. Configure a multi-site listener type (we will create a 2nd listener for the health probe). In the the hostname field, include the name of your web application (e.g. www.contoso.com)

#### *Important - Configure a listener for the Front Door Health Probe

The configuration of Azure Application Gateway as defined above will fail the health probe if the listener is configured for https. Azure Application Gateway does not provision with an ssl certificate and probing the public IP address will fail the SSL Handshake and thus, render the backend offline.

2. Once Application Gateway is successfully deployed, configure a new listner for the Azure Front Door health probe.

    A. Create a new (https) listener for the health probe. Apply your pre-existing multi-domain or wildcard certificate, or upload a new certificate. Configure the listener as multi-site and provide a custom domain name (e.g. appgw.contoso.com)

    B. Add a request routing rule with your listener configured as the front end and your backend target and http settings as defined previously

    C. Create an A record in your public DNS that points to the health probe listener hostname to the public IP address of the Azure Application Gateway

### Azure Front Door Configuration

1. Deploy Azure Front Door in accordance to your standards. Key configuration points.

    A. Configure a globally unique front end (e.g. [name].azurefd.net). Note that you will need to create a CNAME record in your public DNS that represents your web application (e.g. www.contoso.com) which points to the hostname of the Front Door

    B. Add a backend pool, choose the backend type as Application Gateway, navigate to the Appication Gateway object, and clear the configuration on backend host header value (Front Door will pass the request header)

    C. Add a routing rule and provision the Front Door.

#### *Important - Configure a new Front End and reconfigure the backend

2. Add a new Front End\Custom Domain Name

    A. Apply a custom hostname (e.g. www.contoso.com). This requires your CNAME in the step above is configured properly.
    
    B. Configure custom domain https. You can use a Front Door generated certificate or upload your own certificate (requires Key Vault and a PFX file with NO password)

    C. Update the routing rule with the new Front End

    D. Update the backend hostname. By default, Azure Front Door will configure the public IP address of the Azure Applciation Gateway. Unfortnately, this configuration will fail the health probe as the Appliation Gateway does not provision with an SSL certificate (SSL Handshake will fail) and no hostname is configured. Update this value with the hostname of your healthprobe listener (e.g. appgw.contoso.com)

    E. You may need to clear the backend host header value as updating the backend hostname will also update the backend host header. Update and save the configuration. 

### Test!!!

At this point, hopefully everything works! Go to your web URL (e.g. www.contoso.com) and verify functionality. If things are not working right, verify all of the above but specifically make sure the healthprobe listener on the Application Gateway is configured properly as the backend configuration in Azure Front Door. 

I hope you found this helpful, I'm unaware of any documentation that covers this scenario end to end so wanted to document it based on my experience.