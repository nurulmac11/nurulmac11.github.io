+++
title = "Setting SSL for Custom Subdomain on Google App Engine"
date = 2022-11-12
description = "How to configure SSL for custom subdomain for cloudflare/google cloud platform."

[taxonomies]
tags = ["google app engine", "google cloud platform", "cloudflare", "ssl", "subdomain"]
+++

We have 3 separate environments(namely, version) for our app which is hosted in google app engine; development, staging
and production. 
We already have example.com mapped to master version of app but I need to map staging.extra.example.com to staging
environment with ssl enabled.
Google App Engine allows you to add a wildcard custom domain then it automatically maps each versions as subdomains of
it.
So, in our case, if we add *.extra.example.com, google app engine will automatically map staging environment(version) to
staging.extra.example.com.

<!-- more -->

But there is a problem with wildcard subdomains, google app engine's managed SSL certificates doesn't work in this
situation.
Check this quote for explanation;
> "But why?" you will probably ask yourself. To issue a wildcard SSL certificate with LetsEncrypt, you need to achieve a
> DNS-01 challenge instead of HTTP-01 challenge. To automate certificate renewal (which is required given their TTL), you
> need to modify your DNS zone via API, which is different for each DNS provider. That is why Google does not (for now)
> implement managed wildcard SSL certificates.

So, we will need to generate our certificate using certbot. There was a working ready to use google cloud certbot
image (eu.gcr.io/gcloud-certbot/gcloud-certbot)
but it seems like that doesn't work anymore. So, instead of using this image and automating it on cloud, we will
generate certificates on our localhost, then upload it manually.
Since we will need to this once in a every 3 months, don't forget to write a wiki about process for your company.

### Creating SSL Certificate

For this part, you will need cloudflare api token. In this post, I won't explain it but you can find several guides on
web about this.

> 1 - Create a new directory to work within it.

```shell
mkdir ssl && cd ssl
```

> 2 - Create a new file named cloudflare with this command;

```shell
touch cloudflare
```

> 3 - After that fill cloudflare file up to the type of cloudflare key that you got, don't forgot to key or token with
> your credentials.

```shell
# Example credentials file using restricted API Token (recommended):
dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567
```

or

```shell
# Example credentials file using Global API Key (not recommended):
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234
```

> 4 - After that create docker-compose.yml file, don't forget to change website and email;

```yaml
version: "2"

services:
  letsencrypt-cloudflare:
    image: certbot/dns-cloudflare

    # Issue certificate
    command: certonly --non-interactive --dns-cloudflare --dns-cloudflare-credentials /opt/cloudflare/credentials --agree-tos -d extra.example.com --email youremail@example.com --server https://acme-v02.api.letsencrypt.org/directory

    volumes:
      - ./cloudflare:/opt/cloudflare
      - ./letsencrypt:/etc/letsencrypt
      - ./letsencrypt/log:/var/log/letsencrypt
```

> 5 - Then run;

```shell
docker-compose up
```

and you should got an output like this;

```shell
[+] Running 1/1
 ⠿ Container sslproblems-letsencrypt-cloudflare-1  Recreated                                                                                                                             0.1s
Attaching to sslproblems-letsencrypt-cloudflare-1
sslproblems-letsencrypt-cloudflare-1  | Saving debug log to /var/log/letsencrypt/letsencrypt.log
sslproblems-letsencrypt-cloudflare-1  | Account registered.
sslproblems-letsencrypt-cloudflare-1  | Requesting a certificate for extra.example.com
sslproblems-letsencrypt-cloudflare-1  | Unsafe permissions on credentials configuration file: /opt/cloudflare/credentials
sslproblems-letsencrypt-cloudflare-1  | Waiting 10 seconds for DNS changes to propagate
sslproblems-letsencrypt-cloudflare-1  |
sslproblems-letsencrypt-cloudflare-1  | Successfully received certificate.
sslproblems-letsencrypt-cloudflare-1  | Certificate is saved at: /etc/letsencrypt/live/extra.example.com/fullchain.pem
sslproblems-letsencrypt-cloudflare-1  | Key is saved at:         /etc/letsencrypt/live/extra.example.com/privkey.pem
sslproblems-letsencrypt-cloudflare-1  | This certificate expires on 2023-02-10.
sslproblems-letsencrypt-cloudflare-1  | These files will be updated when the certificate renews.
sslproblems-letsencrypt-cloudflare-1  | NEXT STEPS:
sslproblems-letsencrypt-cloudflare-1  | - The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.
sslproblems-letsencrypt-cloudflare-1  |
sslproblems-letsencrypt-cloudflare-1  | - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
sslproblems-letsencrypt-cloudflare-1  | If you like Certbot, please consider supporting our work by:
sslproblems-letsencrypt-cloudflare-1  |  * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
sslproblems-letsencrypt-cloudflare-1  |  * Donating to EFF:                    https://eff.org/donate-le
sslproblems-letsencrypt-cloudflare-1  | - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
sslproblems-letsencrypt-cloudflare-1 exited with code 0
```

So, now you have your ssl certificates. We will upload these to google cloud platform.

### Adding certificates to custom domain

After getting your ssl certificates, which is `letsencrypt/live/extra.example.com/fullchain.pem`
and `letsencrypt/live/extra.example.com/privkey.pem`
you can add these to google app engine.

> 1 - <a href="https://console.cloud.google.com/appengine/settings/certificates" rel="external" target="_blank">
> https://console.cloud.google.com/appengine/settings/certificates</a>
> Go to this url and click on `UPLOAD NEW CERTIFICATE`. So before uploading we need one more step.

> 2 - Since privkey.pem file is incompatible with google, we need to convert it using this command;
>
> `openssl rsa -in privkey.pem > out_privkey.pem`

> 3 - After this, we can upload fullchain to public key section and out_privkey to RSA private key section.

> 4 - Then, refresh the page and click to certificate that you just created. And click add custom domain. Select your
> main domain on the
> `Select the domain you want to use` section and in the `Point your domain to example..` section,
> type `*.extra.example.com`. Then save this record.

> 5 - After that refresh the page, and click on the certificate that you just created, and mark the domain if it's not
> marked automatically to enable this certificate for that domain in the `Enable SSL for the following custom domains`
> section.

> 6 - So as a final step, check `Custom Domains` tab. There should be a new record with your new custom subdomain.
> Add that record to your cloudflare dns records. 

That was all. Hope that works for you too. Reach me out via linkedin if you have any questions.

Resources:

- <a rel="external" target="_blank" href="https://www.nodinrogers.com/post/2022-03-10-certbot-cloudflare-docker">https://www.nodinrogers.com/post/2022-03-10-certbot-cloudflare-docker</a>
- <a rel="external" target="_blank" href="https://www.murrayc.com/permalink/2017/10/15/google-app-engine-using-subdomains/">https://www.murrayc.com/permalink/2017/10/15/google-app-engine-using-subdomains/</a>
- <a rel="external" target="_blank" href="https://dev.to/zenika/how-to-use-secured-wildcard-custom-domains-on-appengine-1a9b">https://dev.to/zenika/how-to-use-secured-wildcard-custom-domains-on-appengine-1a9b</a>
- <a rel="external" target="_blank" href="https://certbot-dns-cloudflare.readthedocs.io/en/stable/#credentials">https://certbot-dns-cloudflare.readthedocs.io/en/stable/#credentials</a>