# How to setup Kanidm and Tailscale

Hi! 

I recently setup [Kandim](https://github.com/kanidm/kanidm) as a Identity Provided ([IdP](https://en.wikipedia.org/wiki/Identity_provider))/Identity Management ([IdM](https://en.wikipedia.org/wiki/Identity_management)) solution, primarily for [Tailscale](https://tailscale.com/). However now it's setup correctly I suppose I can use it for all my [Oauth](https://en.wikipedia.org/wiki/OAuth)/[OpenID](https://en.wikipedia.org/wiki/OpenID) tasks. 

Big thanks [@yaleman](https://github.com/yaleman) for suggesting it to me as a good IdP solution.

Noting there was a bit of learning curve, and some dicking around to get it all working... I figured maybe I should write a guide on how I set it up, to save some other high functioning nerd (my wife would argue otherwise) sometime if they decide also that Google isn't a good trust anchor for all your auth for you home lab.


# Kanidm setup

Eventually I'll flesh this out with the bits about Kanidm setup...

## Kanidm install (Docker)

Eventually I'll detail how I run it in docker here...

## User setup (Kandim Admins)

Eventually I'll detail how to add users to Kandim for Tailscale

## User setup (Tailscale Admins)

Eventually I'll detail how to add users to Kandim for Tailscale

## Group Setup

Eventually I'll detail how to add groups to Kandim for Tailscale

## Oauth Tailscale Config

Eventually I'll detail how to configure Oauth for Kandim with the right scopes configured, and a group that will allow access to this service and assign the right scopes.

# Cloudflare Tunnel Setup

## Cloudflared install (Docker)

Eventually I'll document how I run Cloudflared in docker here...

## Cloudflare Tunnel Config

Eventually I'll document how I run configure Cloudflare tunnel in the console

## WAF Considerations

I spent ages trying to work out why my tunnelled traffic never hit my Kandim server, turns out the WAF thought some of the traffic was a bot. Who knew...

# Webfinger (Apache2.4)

[Tailscale requires a WebFinger](https://tailscale.com/kb/1240/sso-custom-oidc/) with OIDC issuer discovery, see [RFC 7033](https://www.rfc-editor.org/rfc/rfc7033#section-3.1) to be configured in order to be able to configure and authenticate to your Kandim instance. Back in my day finger was a protocol and service you ran (showing my age), so I was somewhat familiar with the concept, however had no idea Webfinger was a thing.

I spent a bit of time unpicking how this was supposed to work and found a few example of how to set it up, most of the examples were to allow custom names on Madtedon.
A couple of references and tools I used:
 - https://www.packetizer.com/ws/webfinger/server.html
 - https://stolley.dev/posted/getting-webfinger-to-play-nicely-on-nginx/
 - https://webfinger.net/lookup/

## Create webfinger folder and files

I created a folder in my web root:

    /var/www/wf/

Within this folder I created a file in the following JRD documents using the following naming format: 

    acct:\$USERNAME@\$FQDN.json

Doing it like this allows for you to host many WebFingers for each different user or service you intend to do something with.

The contents of my *acct:\$USERNAME@\$FQDN.json* file is:
**NOTE:** The FQDN should match the e-mail address FQDN set with your user in Kandim.
**NOTE:** KANDIM_OAUTH_SERVICE_URL us obtained from Kandim cli after the Oauth service is setup

    {
      "subject": "acct:$USERNAME@$FQDN",
      "links": [
        {
          "rel": "http://openid.net/specs/connect/1.0/issuer",
          "href": "$KANDIM_OAUTH_SERVICE_URL"
        }
      ]
    }

Also within the */var/www/wf/* directory I have a .htaccess file containing the following:

    AddType application/json;qs=0.5 .json
    Header set Access-Control-Allow-Origin "*"


## Create Apache2.4 configuration

Now we have the correct JRD json documents and a folder to house them, we need to configure Apache2.4 to allow access to it and do some re-writing and header junk.

I created the following file:

    /etc/apache2/sites-available/webfinger.conf

The file contains the following, noting it's for Apache2.4 other versions will have different authorisation configuration, Google it I guess?

    <Directory /var/www/wf/>
        AddType application/json;qs=0.5 .json
        AllowOverride None
        Require all granted
        Header set Access-Control-Allow-Origin "*"
    </Directory>
    
    <Location "/.well-known/webfinger">
        # Re-write rules
        RewriteEngine On
    
        RewriteCond %{REQUEST_URI}  /\.well-known/webfinger$
        RewriteCond %{QUERY_STRING} resource=([^&]+)
        RewriteRule ^(.*)$ /wf/%1.json [L]
    
        RewriteRule ^/.well-known/webfinger$ - [L,R=400]
    </Location>

Don't forget to enable your new site:

    sudo a2ensite webfinger.conf

And then check everything is in order config wise:

    sudo apachectl -t

And then restart Apache (yes I'm a dirty systemd sellout, I gave up fighting it)

    sudo systemctl reload apache2.service

## Test it works

Visit the https://webfinger.net/lookup/, enter your e-mail address of the user that will log into Tailscale, and if all is good it should return some Json and a log with the full URL of where the Webfinger file is.
