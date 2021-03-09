# mikrotik_EAP-TLS

We are going to do this in the following steps
- Enable CRL
- Create CA and certificates
- Setup wireless AP
- Setup wireless client

### Enable CRL

By default on recent RouterOS versions, CRL is disabled by default. In order to be able to revoce certificates later and effectively bar clients from connecting, CRL on RouterOS has to be eenabled. In our use case, we dont need the CRL download feature.

As a side note, in case you want to use freeradius, you have to enable CRL download and also enable www service, so the freeradius server can download the CRL from RouterOS.

```
/certificate settings
set crl-use=yes
```

