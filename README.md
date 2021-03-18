# mikrotik_EAP-TLS

We are going to do this in the following steps
- Enable CRL
- Create CA and certificates
- Setup wireless AP
- Setup wireless client

We assume RouterOS is on 10.0.0.1 and APs are managed via CAPsMAN. And you are somewhat familiar with Mikrotik stuff.

### Enable CRL

By default on recent RouterOS versions, CRL is disabled. In order to be able to revoke certificates later and effectively bar clients from connecting, CRL on RouterOS has to be eenabled. In our use case, we dont need the CRL download feature.

As a side note, in case you want to use freeradius, you have to enable CRL download and also enable www service, so the freeradius server can download the CRL from RouterOS.

```
/certificate settings
set crl-use=yes
```

### Create CA and certificates

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

### Setup wireless AP

We assume that we already have a working CAPsMAN setup and provisioned the AP with a security profile.

Create a new security profile.

```
name="security_eap-tls" authentication-types=wpa2-eap encryption=aes-ccm group-encryption=aes-ccm eap-methods=eap-tls tls-mode=verify-certificate-with-crl tls-certificate=EAP_AP
```

Now change the respective configuration for the AP to use the new profile and provision.

### Setup wireless client

For Windows, Android and iOS clients, you simply import the following two files

```
cert_export_RouterCA.crt
-> WIN: double click to install and specify to go into trusted root certification authorities (local machine or current user)
-> AND: install network certificate in advanced wireless settings
-> IOS: https://apple.stackexchange.com/questions/326208/how-do-i-configure-an-ipad-to-use-eap-tls

cert_export_EAP_Client.p12
-> WIN: double click to install (current user)
-> AND: as above
-> IOS: as above
```

From here, you can configure the respective device to use your imported CA and certificate. WIN and IOS usually autoselect when trying ot connect. AND needs manual configuration.


### Final remarks

Before deploying for real, make sure you check revokation works as expected, since you cannot delete certificates anymore (unless you remove all of them together with the CA).

Some links that might come in handy:

https://wiki.mikrotik.com/wiki/Manual:Wireless_EAP-TLS_using_RouterOS_with_FreeRADIUS
https://www.nkent.us/wiki/index.php/Wireless_networking_with_CAPsMAN_and_the_MikroTik_cAP_ac
https://serverfault.com/questions/986375/mikrotik-eap-tls-wifi-config-using-certificates
