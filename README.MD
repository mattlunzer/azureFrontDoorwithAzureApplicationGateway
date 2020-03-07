## Azure Front Door WITH Azure Application Gateway Architecture and Configuration Guide

### Purpose

The purpose of this document is to offer a guide for the configuration of two key Azure load balancing services, Azure Front Door AND Azure Application Gateway. Throughout Microsoft documentation you will find design patterns that promote this capability and the benefits it offers.

For example, as of the time of this writing, the URL below illustrates Azure Front Door routing http traffic to Azure Application Gateway which routes http traffic to backend web services. 

- https://docs.microsoft.com/en-us/azure/frontdoor/front-door-lb-with-azure-app-delivery-suite#building-with-azures-application-delivery-suite

However, there is limited how-to documentation to impleemnt this configuration. This guide is intended to account for that.

