<properties linkid="dev-nodejs-enablessl" urlDisplayName="Enable SSL" pageTitle="Configure SSL for a cloud service (Node.js) - Windows Azure" metaKeywords="Node.js Azure SSL, Node.js Azure HTTPS" metaDescription="Learn how to specify an HTTPS endpoint for a Node.js web role and how to upload an SSL certificate to secure your application." metaCanonical="" disqusComments="1" umbracoNaviHide="0" />

<div chunk="../chunks/article-left-menu.md" />

# Configuring SSL for a Node.js Application in a Windows Azure Web Role

Secure Socket Layer (SSL) encryption is the most commonly used method of
securing data sent across the internet. This common task discusses how
to specify an HTTPS endpoint for a Node.js application hosted as a Windows Azure Cloud Service in a web role and how to upload an
SSL certificate to secure your application.

<div class="dev-callout">
	<b>Note</b>
	<p>The steps in this article only apply to node applications hosted as a Windows Azure Cloud Service in a web role.</p>
	</div>

This task includes the following steps:

-   [Step 1: Create a Node.js service and publish the service to the cloud]
-   [Step 2: Get an SSL certificate]
-   [Step 3: Import the SSL certificate]
-   [Step 4: Modify the Service Definition and Configuration Files]
-   [Step 5: Connect to the Role Instance by Using HTTPS]

## <a name="step1"> </a>Step 1: Create a Node.js service and publish the service to the cloud

When a Node.js application is deployed to a Windows Azure web role, the
server certificate and SSL connection are managed by Internet
Information Services (IIS), so that the Node.js service can be written
as if it were an http service. You can create a simple Node.js 'hello
world' service using the Windows Azure PowerShell using these steps:

1. From the **Start Menu** or **Start Screen**, search for **Windows Azure PowerShell**. Finally, right-click **Windows Azure PowerShell** and select **Run As Administrator**.

	![Windows Azure PowerShell icon][powershell-menu]

	<div chunk="../Chunks/install-dev-tools.md" />

2.  Create a new service project using the **New-AzureServiceProject** cmdlet. 

	![][1]

3.  Add a web role to your service using **Add-AzureNodeWebRole** cmdlet:

    ![][2]

4.  Publish your service to the cloud using **Publish-AzureServiceProject** cmdlet:

    ![][3]

	<div class="dev-callout">
	<strong>Note</strong>
	<p>If you have not previously imported publish settings for your Windows Azure subscription, you will receive an error when trying to publish. For information on downloading and importing the publish settings for your subscription, see <a href="https://www.windowsazure.com/en-us/develop/nodejs/how-to-guides/powershell-cmdlets/#ImportPubSettings">How to Use the Windows Azure PowerShell for Node.js</a></p>
	</div>

The **Created Web Site URL** value returned by the **Publish-AzureServiceProject** cmdlet contains the fully qualified domain name for your hosted application. You will need to obtain an SSL certificate for this specific fully qualified domain name and deploy it to Windows Azure.

## <a name="step2"> </a>Step 2: Get an SSL Certificate

To configure SSL for an application, you first need to get an SSL
certificate that has been signed by a Certificate Authority (CA), a
trusted third-party who issues certificates for this purpose. If you do
not already have one, you will need to obtain one from a company that
sells SSL certificates.

The certificate must meet the following requirements for SSL
certificates in Windows Azure:

-   The certificate must contain a private key.
-   The certificate must be created for key exchange (.pfx file).
-   The certificate's subject name must match the domain used to access
    the cloud service. You cannot acquire an SSL certificate for the
    cloudapp.net domain, so the certificate's subject name must match
    the custom domain name used to access your application. For example, __mysecuresite.cloudapp.net__.
-   The certificate must use a minimum of 2048-bit encryption.

## <a name="step3"> </a>Step 3: Import the SSL certificate

Once you have a certificate, install it into the certificate store on your development machine. This certificate will be retrieved and uploaded to Windows Azure as part of your application deployment package based on configuration changes you make in a subsequent step.

<div class="dev-callout">
<strong>Note</strong>
<p>The steps used in this section are based on the Windows 8 version of the Certificate Import Wizard. If you are using a previous version of Windows, the steps listed here may not match the order displayed in the wizard. If this is the case, fully read this section before using the Certificate Import Wizard so that you understand what overall actions must be performed.</p>
</div>

To import the SSL certificate, perform the following steps:

1.   Using Windows Explorer, navigate to the directory where the **.pfx** file containing the certificate is located and then double-click on the certificate. This will display the Certificate Import Wizard.
	
	![certificate wizard][cert-wizard]

2.   In the **Store Location** section, select **Current User** and then click **Next**. This will install the certificate into the certificate store for your user account.

3.   Continue through the wizard, accepting the defaults, until you arrive at the **Private key protection** screen. Here, you must enter the password (if any) for the certificate. You must also select **Mark this key as exportable**. Finally, click **Next**.

	![private key protection][key-protection]

4. Continue through the wizard, accepting the defaults, until the certificate has successfully been installed.

Now you must modify your service definition to reference the certificate you
have installed.

## <a name="step4"> </a>Step 4: Modify the Service Definition and Configuration Files

Your application must be configured to reference the certificate, and an HTTPS
endpoint must be added. As a result, the service definition and service
configuration files need to be updated.

1.  In the service directory, open the service definition file
    (ServiceDefinition.csdef), add a **Certificates** section within the **WebRole** section, and include the following information about the
    certificate:

        <WebRole name="WebRole1" vmsize="ExtraSmall">
        ...
            <Certificates>
                <Certificate name="SampleCertificate" 
                    storeLocation="LocalMachine" storeName="My" />
            </Certificates>
        ...
        </WebRole>

    The **Certificates** section defines the name of the certificate,
    its location, and the name of the store where it is located. Since we installed the certificate to the user certificate store, a value of "My" is used. Other certificate store locations can also be used. See [How to
    Associate a Certificate with a Service] for more information.

2.  In your service definition file, update the http **InputEndpoint** element within the **Endpoints** section to enable HTTPS:

        <WebRole name="WebRole1" vmsize="Small">
        ...
            <Endpoints>
                <InputEndpoint name="Endpoint1" protocol="https" 
                    port="443" certificate="SampleCertificate" />
            </Endpoints>
        ...
        </WebRole>

    All of the required changes to the service definition file have been
    completed, but you still need to add the certificate information to
    the service configuration file.

3.  In your service configuration files
    (**ServiceConfiguration.Cloud.cscfg** and
    **ServiceConfiguration.Local.cscfg**), add the certificate to the
    empty **Certificates** section within the **Role** section,
    replacing the sample thumbprint value below with that of your
    certificate:

        <Role name="WebRole1">
        ...
            <Certificates>
                <Certificate name="SampleCertificate" 
                    thumbprint="9427befa18ec6865a9ebdc79d4c38de50e6316ff" 
                    thumbprintAlgorithm="sha1" />
            </Certificates>
        ...
        </Role>

4.  Refresh your service configuration in the cloud by publishing
    your service again. At the Windows Azure PowerShell
    prompt, type **Publish-AzureServiceProject** from the service directory.

	As part of the publish process, the referenced certificate will be copied from the local certificate store and included in the deployment package.

## <a name="step5"> </a>Step 5: Connect to the Role Instance by Using HTTPS

Now that your deployment is up and running in Windows Azure, you can
connect to it using HTTPS.

1.  In the Management Portal, select your cloud service, then click **Dashboard**.

2. Scroll down and click the link displayed as the **Site URL**:

    ![the site url][site-url]

	<div class="dev-callout">
	<strong>Note</strong>
	<p>If the Site URL displayed in the portal does not specify HTTPS, then you must manually enter the URL in the browser using HTTPS instead of HTTP.</p>
	</div>

3.  A new browser will open and display your web site.

    Your browser will display a lock icon to indicate that it is
    using an HTTPS connection. This also indicates that your application
    has been configured correctly for SSL.

    ![][8]

## Additional Resources

[How to Associate a Certificate with a Service]

[Configuring SSL for a Node.js Application in a Windows Azure Worker Role]

[How to Configure an SSL Certificate on an HTTPS Endpoint]

  [Step 1: Create a Node.js service and publish the service to the cloud]: #step1
  [Step 2: Get an SSL certificate]: #step2
  [Step 3: Import the SSL certificate]: #step3
  [Step 4: Modify the Service Definition and Configuration Files]: #step4
  [Step 5: Connect to the Role Instance by Using HTTPS]: #step5
  [**Windows Azure PowerShell**]: http://go.microsoft.com/?linkid=9790229&clcid=0x409
  [add-certificate-dialog]: ../Media/add-certificate.png
  [add-certificate]: ../Media/no-certificates.png
  [cloud-services]: ../Media/cloud-services.png
  [0]: ../../Shared/Media/azure-powershell-menu.png
  [1]: ../Media/enable-ssl-01.png
  [2]: ../Media/enable-ssl-02.png
  [3]: ../Media/enable-ssl-03.png
  [Windows Azure Management Portal]: http://manage.windowsazure.com
  [4]: ../Media/enable-ssl-04.png
  [5]: ../Media/enable-ssl-05.png
  [How to Associate a Certificate with a Service]: http://msdn.microsoft.com/en-us/library/windowsazure/gg465718.aspx
  [6]: ../Media/enable-ssl-03.png
  [site-url]: ../Media/site-url.png
  [8]: ../Media/enable-ssl-08.png
  [How to Configure an SSL Certificate on an HTTPS Endpoint]: http://msdn.microsoft.com/en-us/library/windowsazure/ff795779.aspx
  [powershell-menu]: ../../Shared/Media/azure-powershell-start.png
  [cert-wizard]: ../Media/certificateimport.png
  [key-protection]: ../Media/exportable.png
  [Configuring SSL for a Node.js Application in a Windows Azure Worker Role]: /en-us/develop/nodejs/common-tasks/enable-ssl-worker-role/
