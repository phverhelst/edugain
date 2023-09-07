# IDP Shibboleth Installation
###test
## Basic system configuration
Installation date 05/09/2023  
ISO Image used : ubuntu-22.04.2-live-server-amd64.iso

- Basic installation server
- openssh enabled
- hostname : shibboleth
- user created serveradmin
- security updates installed

Reference material used :  
https://www.belnet.be/sites/default/files/Procedure%20Shib%204-Jetty%209%20-%20Amazon%20Java%20-%202021.07.pdf

## Installation of the operating system

### Updating and upgrading OS
    sudo su -
    apt update && apt-get upgrade -y --no-install-recommends  
    apt upgrade  
    apt update  
    reboot  

### Installing required packages and MC
    sudo su -
    apt install vim wget gnupg ca-certificates openssl apache2 ntp libservlet3.1-java liblogback-java --no-install-recommends  
    apt install mc

## Installing Amazon Coretto JDK
    sudo su -
    wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add
    apt-get install software-properties-common  
    add-apt-repository 'deb https://apt.corretto.aws stable main'  
    apt-get update; sudo apt-get install -y java-11-amazon-corretto-jdk  
    java -version  
    update-alternatives --config java

### Setting the JAVA_HOME in /etc/environment
add a line to /etc/environment   
```
JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto  
```
    source /etc/profile
    source /etc/environment
    export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto
    echo $JAVA_HOME

## Configuring the host

add a line to /etc/hosts :  
```
<SERVER IP ADDRESS> idp.example.org <HOSTNAME>
```
    sudo su -
    hostnamectl set-hostname <HOSTNAME>
    hostnamectl status

## Installing Shibboleth IDP

    sudo su -
    cd /usr/local/src
    sudo wget https://shibboleth.net/downloads/identity-provider/latest4/shibboleth-identity-provider-4.3.1.tar.gz
    sudo tar -xzf shibboleth-identity-provider-4.3.1.tar.gz
    cd shibboleth-identity-provider-4.3.1/bin

Choose a *Backchannel PKCS12 Password*  
Choose  a *Cookie Encryption Key Password*  
Attribute Scope must be <example.org> by default  

### Installing Jetty 9 Web Server

    sudo su -
    cd /usr/local/src
    wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-distribution/9.4.42.v20210604/jetty-distribution-9.4.42.v20210604.tar.gz
    tar xzvf jetty-distribution-9.4.42.v20210604.tar.gz
    ln -nsf jetty-distribution-9.4.42.v20210604 jetty-src
    mkdir /opt/jetty
    wget https://registry.idem.garr.it/idem-conf/shibboleth/IDP4/jetty/start.ini -O /opt/jetty/start.ini
    mkdir /opt/jetty/tmp
    chown jetty:jetty /opt/jetty/tmp
    chown -R jetty:jetty /opt/jetty /usr/local/src/jetty-src
    mkdir /var/log/jetty
    mkdir /opt/jetty/logs
    chown jetty:jetty /var/log/jetty /opt/jetty/logs

Create /etc/default/jetty and insert :  
```
JETTY_HOME=/usr/local/src/jetty-src
JETTY_BASE=/opt/jetty  
JETTY_USER=jetty  
JETTY_START_LOG=/var/log/jetty/start.log  
TMPDIR=/opt/jetty/tmp  
```
    sudo su -
    ln -s /usr/local/src/jetty-src/bin/jetty.sh jetty
    update-rc.d jetty defaults
    service jetty check
    service jetty start

### Configuring Jetty

    sudo mkdir /opt/jetty/webapps

Create /opt/jetty/wabapps/idp.xml and insert :  
```
<Configure class="org.eclipse.jetty.webapp.WebAppContext">  
<Set name="war"><SystemProperty name="idp.home"/>/war/idp.war</Set>  
 <Set name="contextPath">/idp</Set>  
 <Set name="extractWAR">false</Set>  
 <Set name="copyWebDir">false</Set>  
 <Set name="copyWebInf">true</Set>  
 <Set name="persistTempDirectory">false</Set>  
</Configure>  
```
    sudo su -
    cd /opt/shibboleth-idp
    chown -R jetty logs/ metadata/ credentials/ conf/ war/
    systemctl restart jetty.service
    service jetty check
    service jetty start
    service jetty check

### Configuring Apache2 and SSL

    sudo su -
    mkdir /var/www/html/$(hostname -f)
    chown -R www-data: /var/www/html/$(hostname -f)
    touch /var/www/html/$(hostname -f)/index.html
    echo '<h1>It Works!</h1>' | sudo tee /var/www/html/$(hostname -f)/index.html
    wget https://registry.idem.garr.it/idem-conf/shibboleth/IDP4/apache2/idp.example.org.conf -O /etc/apache2/sites-available/$(hostname -f).conf

Adjust the content $(hostname -f).conf

#### Create request file for public certificate

    sudo touch /etc/ssl/$(hostname -f).conf

Insert in $(hostname -f).conf :
```
    [req]
    default_bits = 2048
    prompt = no
    encrypte_key = no
    default_md = sha256
    distinguished_name = dn

    [dn]
    C = country
    O = organisation
    CN = idp.example.org
```

    sudo openssl req -new -config /etc/ssl/$(hostname -f).conf -keyout /etc/ssl/private/$(hostname -f).key -out /etc/ssl/$(hostname -f).csr

Use the csr to obtain a public certificate, copy the crt file in /etc/ssl/certs/ folder.

    sudo su -
    chmod 400 /etc/ssl/private/$(hostname -f).key
    chmod 644 /etc/ssl/certs/$(hostname -f).crt
    a2enmod proxy_http ssl headers alias include negotiation
    systemctl restart apache2
    systemctl status apache2
    a2ensite $(hostname -f).conf
    systemctl reload apache2

Testing IDP access at https://idp.example.org/idp/shibboleth  
Testing the site SSL configuration at https://www.ssllabs.com/ssltest/analyze.html

## Configuring Shibboleth Identity Provider

Remove validUntil="2021-06-30T12:15:34.029Z" or equivalent in /opt/shibboleth/metadata  
Testing the idp installation with /opt/shibboleth-idp/bin/status.sh  

### Autoriser l'accès à la page https://idp.example.org/status

Edit /opt/shibboleth-idp/conf/access-control.xml  to add the ip addresse,  in CIDR format at the line:  

    p:allowedRanges="#{ {'127.0.0.1/32', '::1/128', **'x.x.x.x/x'**} }" />  

Reload the configuration of the idp:  

    /opt/shibboleth-idp/bin/reload-service.sh -id shibboleth.ReloadableAccessControlService  

### Configure AD Connection

#### Installing the ldap-utils

    sudo su -
    apt install ldap-utils

#### Testing the ldap non secure connexion with an authorized account from the AD.

Create an idpuser in the Active Directory.  
Use ldapsearch to test the connexion.  
Example idpuser is used for searching a user John Doe with johndoe as sAMAccountName, the two accounts are is the Users OU.

    ldapsearch -x -D "CN=idpuser,CN=Users,DC=example,DC=org" -H ldap://dc.example.org -s sub "sAMAccountName=johndoe" -W
    
#### Configuring and testing the ldap SSL and TLS connexion

Assume the prerequisite of a PKI infrastructure and deployment of certificates for the domain controllers and clients.
Testing on the domain controller with ldp.exe the connexion in ssl (check SSL and choose port 636)  
Place Root and intermediate certificates of the Active Directory infrastructure in /usr/local/share/ca-certificates and update :

    sudo update-ca-certificates


From the idp retrieving of the certificates from the domain controller can be achieved with openssl s_client :

    sudo su -
    openssl s_client -showcerts -verify 5 -connect dc.exemple.org:636  < /dev/null | awk '/BEGIN/,/END/{ if(/BEGIN/)    {a++}; out="dc-cert"a".pem"; print >out}'
    
Certificates can be retrieved from the certificate store on the DC, trough the Certificates mmc, local computer store, files must be exported in Base-64 encoded X.509.

Edit the /etc/ldap.conf to be able to test the connexion with ldapsearch.  
Add the path to the root certificate of the Active Directory PKI infrastructure.

    TLS_CACERT  /usr/local/share/ca-certificates/root.example.org.crt


Example idpuser is used for searching a user John Doe with johndoe as sAMAccountName, the two accounts are is the Users OU, TLS then SSL connexion.

    ldapsearch -x -D "CN=idpuser,CN=Users,DC=example,DC=org" -H ldap://dc.example.org -s sub "sAMAccountName=johndoe" -W -Z
    ldapsearch -x -D "CN=idpuser,CN=Users,DC=example,DC=org" -H ldap://dc.example.org:636 -s sub "sAMAccountName=johndoe" -W 






### Add SAML1 and SAML2 support

Edit /opt/shibboleth-idp/conf/relying-party.xml and uncomment the SAML1 and SAML2 in <bean id="shibboleth.DefaultRelyingParty parent="RelyingParty">

    <bean id="shibboleth.DefaultRelyingParty" parent="RelyingParty">
            <property name="profileConfigurations">
                <list>
                    <ref bean="Shibboleth.SSO" />
                    <ref bean="SAML1.AttributeQuery" />
                    <ref bean="SAML1.ArtifactResolution" />
                    <ref bean="SAML2.SSO" />
                    <ref bean="SAML2.ECP" />
                    <ref bean="SAML2.Logout" />
                    <ref bean="SAML2.AttributeQuery" />
                    <ref bean="SAML2.ArtifactResolution" />
                    <ref bean="Liberty.SSOS" />
                </list>
        </property>
    </bean>






## Configuring Belnet Federation

###

    sudo wget https://federation.belnet.be/certificate.federation.belnet.be.pem -O /opt/shibboleth-idp/credentials/certificate.federation.belnet.be.crt




## References

https://www.ibm.com/support/pages/how-test-ca-certificate-and-ldap-connection-over-ssltls  
https://support.kerioconnect.gfi.com/hc/en-us/articles/360015200119-Adding-Trusted-Root-Certificates-to-the-Server


