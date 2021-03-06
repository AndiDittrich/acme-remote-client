acme-remote-client
=====================================
This is an extensions of [acme-tiny](https://github.com/diafygi/acme-tiny), a small and simple audtiable python script that you can use to issue and renew [Let's Encrypt](https://letsencrypt.org/) certificates.
It can talk to any [ACME](https://letsencrypt.github.io/acme-spec/) based web services to issues [X509v3](https://en.wikipedia.org/wiki/X.509) certificates based on your CSR.

Features
--------------------------------------

  * **CSR in, CERT out - no less, no more**
  * Intended to use with [Let's Encrypt](https://letsencrypt.org/)
  * Ability to run on a dedicated Certificate Management Host
  * Upload hook to transfer the http-challenge-token
  * Domain Validation via [SimpleHTTP/http-01](https://letsencrypt.github.io/acme-spec/#simple-http) challenge
  * Runs as normal user - no root/sudo required
  * Easily to integrate into existing PKI management environments

Audience
--------------------------------------

This derived version is targeted to people who are familar with **Let's Encrypt, X509 certificate management (PKI), webserver administration (nginx, apache, lighttpd, ..) and server management!**
It's a small tool which can be integrated into your existing toolchain to manage X509 certificates.
It does only talk to the [Let's Encrypt Authority](https://letsencrypt.org/) to validate your domain name and issue a certificate.

The webserver management is **on yours!** You have to create your CSR by yourself and install the generated certificates manually.

**If you plan to use this tool on a single server, please take a look on the [original vesion](https://github.com/diafygi/acme-tiny) before using this extended one!**

**If you are a beginner or don't understand what I just said, this script likely isn't for you! Please use the official [Let's Encrypt client](https://github.com/letsencrypt/letsencrypt)**

Preface - Systems Architecture
--------------------------------------

This is a small example how the certificate management can be outsourced to an additional host.

In this example, requests from the Let's Encrypt servers are redirected to a single Frontend-Webserver which serves your domain.
The server will catch all requests to `mysite.com/.well-known/acme-challenge/*` and redirects them to `/var/www/acme-challenges`.
This special folder is remote-accessible via SSH: domain validation tokens can be pushed by a trusted remote host!

The big benefit of such a solution is, that you need only one (virtual) host which is responsible for your Certificate Management and deployment.
It's especially required when using a bunch of webservers - such situation is no suitable to handle with the official Let's Encrypt client!

```
  Internet
      +
      |
      |
      v
+------------+        +------------------------------------------+
|Firewall    |        |Frontend-Webserver (serves mysite.com)    |
|IDPS        +------> |Requests to /.well-known/acme-challenge/* |
|LoadBalancer|        |are redirected to /var/www/acme/          |
+------------+        +------------------------------------------+
                                      ^
                                      |
                                      |  Secure SSH Connection
                                      |  to upload the challenge
                                      |
                     +----------------+-------------------------+
                     |Cert Management Host (with acme-tiny.py)  |
                     | - manage keys, csr, certs                |
                     | - requests Let's Encrypt Cert            |
                     | - Upload the Challenge                   |
                     +------------------------------------------+
```

Usage
--------------------------------------

## Step 1: Create a Let's Encrypt account private key (if you haven't already)

You must have a public key registered with Let's Encrypt and sign your requests with the corresponding private key.
To accomplish this you need to initially create a key, that can be used by acme-tiny, to register a account for you and sign all following requests.

```shell
# create a new 4096bit RSA Keypair
openssl genrsa -out .letsencrypt/account.pem 4096
```

## Step 2: Create or Provide CSR (Certificate Signing Request)

I assume that you are familar with this procedure. The tool requires the CSR to be in **DER** format.

**Example: Convert PEM to DER**

```shell
openssl req -in mysite_com.csr -outform DER -out mysite_com.der
```

## Step 3: Prepare your Webserver

If you plan to run the tool on a dedicated management host, you have to prepare your webserver to catch the validation requests.
The SimpleHTTP validation of the ACME protocol is using the static path `yourdomain.com/.well-known/acme-challenge/*` to catch single files including a token.
It is recommended to redirect this path to a **central location** on your webserver, e.g. `/var/www/acme-challenges`

#### Lighttpd ####

With lighttpd, you can use [mod_alias](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModAlias) to redirect the request:

```conf
alias.url += ( "/.well-known/acme-challenge" => "/var/www/acme-challenge/" )
```

#### NGINX ####

```conf
server{
  location '/.well-known/acme-challenge' {
    default_type "text/plain";
    root /var/www/acme-challenge;
  }
}
```

## Step 4: Run the Tool

#### On a single webserver ####

**Assumption**: Your Webserver catches all http-challenges on the requested domains `/.well-known/acme-challenge/*` path and redirects them to `/var/www/acme-challenges`

**Request the Certificate**

```shell
python acme_remote_client.py \
  --account-key ./.letsencrypt/account.pem \
  --csr ./requests/mysite_com.der \
  --acme-dir /var/www/acme-challenges \
  --out ./certs/mysite_com.cert
```

#### On a dedicated management server ####

**Assumptions:**

   * Your Remote Frontend Webserver catches all http-challenges `/.well-known/acme-challenge/*` and redirects them to `/var/www/acme-challenges`
   * You've created an SSH account (key based auth) which have write access to remote-machines directory `/var/www/acme-challenges`

**Token Upload Script Example**

```shell
#!/usr/bin/env bash

# Upload ACME Token Challenge to Validation Host
token_upload(){
    scp -q \
    -i your_server_ssh_key.pem \
    .challenges/$1 \
    tokenuser@yourserver:/var/www/acme-challenges/$1
}

# Command Dispatching - the first argument will always be "token"
case "$1" in
  token)
    token_upload $2
    ;;
  *)
    echo "Usage: $0 {token} [filename..]"
esac
```

**Request the Certificate**

```shell
python acme_remote_client.py \
  --account-key ./.letsencrypt/account.pem \
  --csr ./requests/mysite_com.der \
  --acme-dir ./.challenges \
  --out ./certs/mysite_com.cert \
  --token-upload ./token-upload.sh
```

## Step 5: Install the certificates

Finally you have to install the certificate manually on your Webserver.

License
-------

**acme-remote-client** is OpenSource and licensed under the Terms of [The MIT License (X11)](http://opensource.org/licenses/MIT). You're welcome to contribute!

The original [acme-tiny](https://github.com/diafygi/acme-tiny) is created by [Daniel Roesler](https://github.com/diafygi) - Many thanks!
