# Docker - Generate Certificate dengan Certbot

Jika `docker-compose` belum tersedia, silahkan install docker-compose terlebih dahulu
```
$ sudo curl -L "https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-$(uname -s)-$(uname -m)"  -o /usr/local/bin/docker-compose
$ sudo mv /usr/local/bin/docker-compose /usr/bin/docker-compose
$ sudo chmod +x /usr/bin/docker-compose
```
Release `docker-compose` https://github.com/docker/compose/releases

```
$ docker-compose --version
Docker Compose version v2.24.7
```

## Create the required directories
```
$ mkdir -p etc-letsencrypt var-lib-letsencrypt var-log-letsencrypt
```

## Docker compose
`docker-compose.yaml`
```
---
version: "3.8"

services:
  certbot:
    image: certbot/certbot
    volumes:
      - ./etc-letsencrypt:/etc/letsencrypt
      - ./var-lib-letsencrypt:/var/lib/letsencrypt
      - ./var-log-letsencrypt:/var/log/letsencrypt
```
mencoba menjalankan cerbot `--help`
```
$ docker-compose run certbot --help
[+] Creating 1/0
 ✔ Network cerbot_default  Created 
... 
certbot [SUBCOMMAND] [options] [-d DOMAIN] [-d DOMAIN] ...

Certbot can obtain and install HTTPS/TLS/SSL certificates.  By default,
it will attempt to use a webserver both for obtaining and installing the
certificate. The most common SUBCOMMANDS and flags are:
...
```

## Create certificate dengan validasi domain
Sesuaikan `example.com` dengan `domain.com` anda
```
$ docker-compose run certbot certonly -d example.com \
--manual --preferred-challenges dns --dry-run
```

Silahkan mengikuti petunjuk dan membuat `DNS TXT record` pada domain
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): admin@example.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.3-September-21-2022.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
Account registered.
Simulating a certificate request for example.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name:

_acme-challenge.example.com.

with the following value:

efYfq-O_G6OHWH_QtoEEByrg7BSLwE0az1FtLXFQEGY

Before continuing, verify the TXT record has been deployed. Depending on the DNS
provider, this may take some time, from a few seconds to multiple minutes. You can
check if it has finished deploying with aid of online tools, such as the Google
Admin Toolbox: https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.example.
Look for one or more bolded line(s) below the line ';ANSWER'. It should show the
value(s) you've just added.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
The dry run was successful.
```

Jika sudah berhasil, saatnya membuat certificate dengan menghilangkan *flag* `--dry-run`

```
$ docker-compose run certbot certonly -d example.com \
--manual --preferred-challenges dns
```
```
...
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/example.com/privkey.pem
This certificate expires on 2024-06-06.
These files will be updated when the certificate renews.

NEXT STEPS:
- This certificate will not be renewed automatically. Autorenewal of --manual certificates requires the use of an authentication hook script (--manual-auth-hook) but one was not provided. To renew this certificate, repeat this same certbot command before the certificate's expiry date.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Certificate berhasil dibuat
```
.
├── docker-compose.yaml
├── etc-letsencrypt
│   ├── accounts
│   │   └── acme-v02.api.letsencrypt.org
│   │       └── directory
│   │           └── 23f29bb2f640affcd10ac21d23684b14
│   │               ├── meta.json
│   │               ├── private_key.json
│   │               └── regr.json
│   ├── archive
│   │   └── example.com
│   │       ├── cert1.pem
│   │       ├── chain1.pem
│   │       ├── fullchain1.pem
│   │       └── privkey1.pem
│   ├── live
│   │   ├── README
│   │   └── example.com
│   │       ├── cert.pem -> ../../archive/example.com/cert1.pem
│   │       ├── chain.pem -> ../../archive/example.com/chain1.pem
│   │       ├── fullchain.pem -> ../../archive/example.com/fullchain1.pem
│   │       ├── privkey.pem -> ../../archive/example.com/privkey1.pem
│   │       └── README
│   ├── renewal
│   │   └── example.com.conf
│   └── renewal-hooks
│       ├── deploy
│       ├── post
│       └── pre
├── var-lib-letsencrypt
│   └── backups
└── var-log-letsencrypt
    ├── letsencrypt.log
    ├── letsencrypt.log.1
    └── letsencrypt.log.2
```