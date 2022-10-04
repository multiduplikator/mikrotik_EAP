# mikrotik EAP-TLS and EAP-PEAP (ROS6 classic or ROS7 with User Manager V5) 

We are going to do this in the following steps
- [Step 1: ROS6 and ROS7](#step-1-ros6-and-ros7)
  - [Enable CRL](#enable-crl)
- [Step 2a: ROS6](#step-2a-ros6)
  - [Create CA and certificates](#ros6---create-ca-and-certificates)
  - [Setup wireless AP](#ros6---setup-wireless-ap)
- [Step 2b: ROS7](#step-2b-ros7)
  - [Create CA and certificates](#ros7---create-ca-and-certificates)
  - [Setup User Manager](#ros7---setup-usermanager)
  - [Setup wireless AP](#ros7---setup-wireless-ap)
- [Step 3: ROS6 and ROS7](#step-3-ros6-and-ros7)
  - [Setup wireless client with EAP-TLS](#setup-wireless-client-with-eap-tls)
  - [Setup wireless client with EAP-PEAP](#setup-wireless-client-with-eap-peap)


We assume RouterOS is on 10.0.0.1 and APs are managed via CAPsMAN. And you are somewhat familiar with Mikrotik stuff.

## Step 1: ROS6 and ROS7

### Enable CRL

By default on recent RouterOS versions, CRL is disabled. In order to be able to revoke certificates later and effectively bar clients from connecting, CRL on RouterOS has to be enabled. In our use case, we dont need the CRL download feature.

As a side note, in case you want to use freeradius, you have to enable www service, so the freeradius server can download the CRL from RouterOS. Inspect a certificate to find the exact download URL to use. If you want RouterOS itself to download external CRLs, you have to enable the CRL Download feature.

```
/certificate settings
set crl-use=yes
```

Now, continue either with Step 2a or with Step 2b.

## Step 2a: ROS6

This setup gives us EAP-TLS only. EAP-PEAP has to be implemented with a sidecar radius server like freeradius (see Final Remarks). You might want to consider to split the wireless networks into one that does EAP-TLS and another one that does EAP passthrough to e.g. freeradius which does the EAP-PEAP.

### ROS6 - Create CA and certificates

This can also be done outside of RouterOS, but this way it is pretty convenient.

```
/certificate add name=RouterCA common-name=Router subject-alt-name=IP:10.0.0.1 key-size=4096 days-valid=3650 key-usage=crl-sign,key-cert-sign
/certificate sign RouterCA ca-crl-host=10.0.0.1 name=RouterCA
# for import as wireless network certificate or trusted root certification authority on client
/certificate export-certificate RouterCA type=pem

/certificate add name=EAP_AP common-name=EAP_AP subject-alt-name=IP:10.0.0.1 key-size=4096 days-valid=3650 key-usage=digital-signature,key-encipherment,tls-server
/certificate sign EAP_AP ca=RouterCA name=EAP_AP
/certificate set EAP_AP trusted=yes
# no export needed

/certificate add name=EAP_Client common-name=EAP_Client key-size=4096 days-valid=3650 key-usage=tls-client
/certificate sign EAP_Client ca=RouterCA name=EAP_Client
/certificate set EAP_Client trusted=yes
# for import as wireless network certificate or user certificate on client
/certificate export-certificate EAP_Client type=pkcs12 export-passphrase=<your_long_passphrase_goes_here>
```

The exported files should be:

```
cert_export_RouterCA.crt -> for deploying the client
cert_export_EAP_Client.p12 -> for deploying the client
```

### ROS6 - Setup wireless AP

We assume that we already have a working CAPsMAN setup and provisioned the AP with a security profile.

Create a new security profile.

```
/caps-man/security add name="security_eap-tls" authentication-types=wpa2-eap encryption=aes-ccm group-encryption=aes-ccm eap-methods=eap-tls tls-mode=verify-certificate-with-crl tls-certificate=EAP_AP
```

Now change the respective configuration for the AP to use the new profile and provision.

## Step 2b: ROS7

This setup gives us EAP-TLS and EAP-PEAP and it can be done on a given wireless network simultaneously. It is crucial to use secp384r1 and sha384 with the certificates. Otherwise, you will run into trouble with Android devices trying to use the anonymous identity, causing a user not found error in Usermanager (at least on ROS 7.1.1).

### ROS7 - Create CA and certificates

Again, this can also be done outside of RouterOS, but this way it is pretty convenient. For the client certificates, it seems that the validity period should not be more than 825 days.

```
# Generating a Certificate Authority
/certificate
add name=RouterCA common-name=Router subject-alt-name=IP:10.0.0.1 key-size=secp384r1 digest-algorithm=sha384 days-valid=1825 key-usage=key-cert-sign,crl-sign
sign RouterCA ca-crl-host=10.0.0.1 name=RouterCA

# Generating a server certificate for User Manager
add name=EAP_AP common-name=EAP_AP subject-alt-name=IP:10.0.0.1 key-size=secp384r1 digest-algorithm=sha384 days-valid=730 key-usage=tls-server
sign EAP_AP ca=RouterCA name=EAP_AP
set EAP_AP trusted=yes

# Generating a client certificate
add name=EAP_Client common-name=EAP_Client days-valid=730 key-size=secp384r1 digest-algorithm=sha384 key-usage=tls-client 
sign EAP_Client ca=RouterCA name=EAP_Client
set EAP_Client trusted=yes

# Exporting the public key of the CA as well as the generated client private key and certificate for distribution to client device
export-certificate RouterCA
export-certificate EAP_Client type=pkcs12 export-passphrase=<your_long_passphrase_goes_here>
```

The exported files should be:

```
cert_export_RouterCA.crt -> for deploying the client
cert_export_EAP_Client.p12 -> for deploying the client
```

### ROS7 - Setup Usermanager

The User Manger serves as the radius server that does the EAP stuff, using the EAP_AP certificate. We enable the router to do radius auth, adjust the user groups to provide one for EAP-TLS and another one for EAP-PEAP. Users with certificate go into the first group, having the same name as the common name of their certificate. Users that do EAP-PEAP go into the second group. Also, we allow for more than one device/connection to share a given user.

```
# Enabling User Manager and specifying, which certificate to use
/user-manager
set enabled=yes certificate=EAP_AP

# Adding access points
/user-manager router
add name=Router address=10.0.0.1 shared-secret=<your_long_shared_secret_goes_here>
# Limiting allowed authentication methods
/user-manager user group
set [find where name=default] outer-auths=eap-peap inner-auths=peap-mschap2
add name=certificate-authenticated outer-auths=eap-tls
# Adding users
/user-manager user
add name=EAP_Client group=certificate-authenticated shared-users=3
add name=SomeUser group=default password=<users_password_goes_here> shared-users=2
```

### ROS7 - Setup wireless AP

We assume that we already have a working CAPsMAN setup and provisioned the AP with a security profile. In ROS6 we did EAP direct against the certificate store. Now, in ROS7, we do it as passthrough to the User Manager. 

To make the passthrough work, we have to create an entry in the RADIUS configuration, that links to the User Manager

```
/radius add address=10.0.0.1 secret=<your_long_shared_secret_goes_here> service=wireless timeout=1s
```

Now, create a new security profile that does the actual passthrough.

```
/caps-man/security add name="security_eap-tls" authentication-types=wpa2-eap encryption=aes-ccm group-encryption=aes-ccm eap-methods=passthrough
```

Now change the respective configuration for the AP to use the new profile and provision.

## Step 3: ROS6 and ROS7

### Setup wireless client with EAP-TLS

For Windows (WIN), Android (AND) and iOS clients, you simply import the following two files

```
cert_export_RouterCA.crt
-> WIN: double click to install and specify to go into trusted root certification authorities (current user or local machine)
-> AND: install network certificate in advanced wireless settings
-> IOS: https://apple.stackexchange.com/questions/326208/how-do-i-configure-an-ipad-to-use-eap-tls

cert_export_EAP_Client.p12
-> WIN: double click to install (current user)
-> AND: as above with .crt
-> IOS: as above with .crt
```

From here, you can configure the respective device to use your imported CA and certificate. WIN and IOS usually autoselect when trying to connect. AND needs manual configuration.

### Setup wireless client with EAP-PEAP

In case you have ROS6, you have to implement EAP-PEAP via passthrough to a sidecar radius server, e.g. freeradius. This works with a dedicated wireless network that does the passthrough.

In case you have ROS7 and User Manager, you can do EAP-PEAP on the same wireless network, using the user/password combination as created in the default group above, i.e. username as "SomeUser" and password as "<users_password_goes_here>" - leaving the anonymous_identity blank.

This mode is particularly usefull for Chromebooks that are remote administered, and where you cannot install certificates permanently.

### Sidenote on WPA3 and Android 11+
We are on the advent of WPA3 and Android 11+ now starts to enforce section 5.1:
"The STA is configured with EAP credentials that explicitly specify a CA root certificate that matches the root certificate in the received Server Certificate message and, if the EAP credentials also include a domain name (FQDN or suffix-only), it matches the domain name (SubjectAltName DNSName if present, otherwise SubjectName CN) of the certificate [2] in the received Server Certificate message."

In somewhat simpler terms, this reads:
"The new Domain field in the wifi config dialog must be the CN or subjectAlternateName of the server certificate." 

Hence (esp. when setting up Androind 11+ wireless clients), make sure that you use the CN of the EAP_AP certificate as Domain field entry - in our scenario that would be "EAP_AP" - when setting up the client. Alternatively, you can add an additional "subject-alt-name=DNS:eap.ap.local" when creating the EAP_AP certificate and respectively use "eap.ap.local" as Domain field entry.

## Final remarks

Before deploying for real, make sure you check revokation works as expected, since you cannot delete certificates anymore (unless you remove all of them together with the CA).

It is relatively simple to exted the above to also provide EAP-TTLS support. ROS6 again via sidecar radius server, ROS7 via the Usermanager and its groups settings inner-auth/outer-auth.

Be aware, that you might struggle to use eapol_test with the ROS7 appraach. At least I did not spent enough time to get it to work with the secp384r1 being a 384-bit prime field Weierstrass curve. 

Some links that might come in handy:

- ROS7
  - https://help.mikrotik.com/docs/display/ROS/Enterprise+wireless+security+with+User+Manager+v5

- ROS6
  - https://wiki.mikrotik.com/wiki/Manual:Wireless_EAP-TLS_using_RouterOS_with_FreeRADIUS
  - https://www.nkent.us/wiki/index.php/Wireless_networking_with_CAPsMAN_and_the_MikroTik_cAP_ac
  - https://serverfault.com/questions/986375/mikrotik-eap-tls-wifi-config-using-certificates
