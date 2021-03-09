# mikrotik_EAP-TLS

We are going to do this in the following steps
- Enable CRL
- Create CA and certificates
- Setup wireless AP
- Setup wireless client

We assume RouterOS is on 10.0.0.1 and APs are managed via CAPsMAN.

### Enable CRL

By default on recent RouterOS versions, CRL is disabled by default. In order to be able to revoce certificates later and effectively bar clients from connecting, CRL on RouterOS has to be eenabled. In our use case, we dont need the CRL download feature.

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
/certificate export-certificate EAP_Client type=pkcs12 export-passphrase=certificate123
```

The exported files should be:
cert_export_RouterCA.crt -> only needed for deploying the client
cert_export_EAP_Client.p12 -> only needed for deploying the client


