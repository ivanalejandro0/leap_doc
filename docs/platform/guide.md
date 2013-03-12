@title = "LEAP Platform Guide"
@nav_title = "Guide"

Services
================================

Every node has one or more services that determines the node's function within your provider's infrastructure.

When adding a new node to your provider, you should ask yourself four questions:

* **many or few?** Some services benefit from having many nodes, while some services are best run on only one or two nodes.
* **required or optional?** Some services are required, while others can be left out.
* **who does the node communicate with?** Some services communicate very heavily with other particular services. Nodes running these services should be close together.
* **public or private?** Some services communicate with the public internet, while others only need to communicate with other nodes in the infrastructure.

Brief overview of the services:

![services diagram](service-diagram.png)

* **webapp**: The web application. Runs both webapp control panel for users and admins as well as the REST API that the client uses. Needs to communicate heavily with `couchdb` nodes. You need at least one, good to have two for redundancy. The webapp does not get a lot of traffic, so you will not need many.
* **couchdb**: The database for users and user data. You can get away with just one, but for proper redundancy you should have at least three. Communicates heavily with `webapp` and `mx` nodes.
* **soledad**: Handles the data syncing with clients. Typically combined with `couchdb` service, since it communicates heavily with couchdb. (not currently in stable release)
* **mx**: Incoming and outgoing MX servers. Communicates with the public internet, clients, and `couchdb` nodes. (not currently in stable release)
* **openvpn**: OpenVPN gateway for clients. You need at least one, but want as many as needed to support the bandwidth your users are doing. The `openvpn` nodes are autonomous and don't need to communicate with any other nodes. Often combined with `tor` service.

Not pictured:

* **monitor**: Internal service to monitor all the other nodes. Currently, you can have zero or one `monitor` nodes.
* **tor**: Sets up a tor exit node, unconnected to any other service.
* **dns**: Not yet implemented.

Certificates
================================

Configuration options
-------------------------------------------

The `ca` option in provider.json provides settings used when generating CAs and certificates. The defaults are as follows:

    "ca": {
      "name": "= global.provider.ca.organization + ' Root CA'",
      "organization": "= global.provider.name",
      "organizational_unit": "= 'https://' + global.provider.name",
      "bit_size": 4096,
      "digest": "SHA256",
      "life_span": "10y",
      "server_certificates": {
        "bit_size": 2024,
        "digest": "SHA256",
        "life_span": "1y"
      },
      "client_certificates": {
        "bit_size": 2024,
        "digest": "SHA256",
        "life_span": "2m",
        "limited_prefix": "LIMITED",
        "unlimited_prefix": "UNLIMITED"
      }
    }

To see what values are used for your provider, run `leap inspect provider.json`. You can modify the defaults as you wish by adding the values to provider.json.

NOTE: A certificate `bit_size` greater than 2024 will probably not be recognized by most commercial CAs.

Certificate Authorities
-----------------------------------------

There are three x.509 certificate authorities (CA) associated with your provider:

1. **Commercial CA:** It is strongly recommended that you purchase a commercial cert for your primary domain. The goal of platform is to not depend on the commercial CA system, but it does increase security and usability if you purchase a certificate. The cert for the commercial CA must live at `files/cert/commercial_ca.crt`.
2. **Server CA:** This is a self-signed CA responsible for signing all the **server** certificates. The private key lives at `files/ca/ca.key` and the public cert lives at `files/ca/ca.crt`. The key is very sensitive information and must be kept private. The public cert is distributed publicly.
3. **Client CA:** This is a self-signed CA responsible for signing all the **client** certificates. The private key lives at `files/ca/client_ca.key` and the public cert lives at `files/ca/client_ca.crt`. Neither file is distribute publicly. It is not a big deal if the private key for the client CA is compromised, you can just generate a new one and re-deploy.

To generate both the Server CA and the Client CA, run the command:

    leap cert ca

Server certificates
-----------------------------------

Most every server in your service provider will have a x.509 certificate, generated by the `leap` command using the Server CA. Whenever you modify any settings of a node that might affect it's certificate (like changing the IP address, hostname, or settings in provider.json), you can magically regenerate all the certs that need to be regenerated with this command:

    leap cert update

Run `leap help cert update` for notes on usage options.

Because the server certificates are generated locally on your personal machine, the private key for the Server CA need never be put on any server. It is up to you to keep this file secure.

Client certificates
--------------------------------

Every leap client gets its own time-limited client certificate. This cert is use to connect to the OpenVPN gateway (and probably other things in the future). It is generated on the fly by the webapp using the Client CA.

To make this work, the private key of the Client CA is made available to the webapp. This might seem bad, but compromise of the Client CA simply allows the attacker to use the OpenVPN gateways without paying. In the future, we plan to add a command to automatically regenerate the Client CA periodically.

There are two types of client certificates: limited and unlimited. A client using a limited cert will have its bandwidth limited to the rate specified by `provider.service.bandwidth_limit` (in Bytes per second). An unlimited cert is given to the user if they authenticate and the user's service level matches one configured in `provider.service.levels` without bandwidth limits. Otherwise, the user is given a limited client cert.

Commercial certificates
-----------------------------------

We strongly recommend that you use a commercial signed server certificate for your primary domain (in other words, a certificate with a common name matching whatever you have configured for `provider.domain`). This provides several benefits:

1. When users visit your website, they don't get a scary notice that something is wrong.
2. When a user runs the leap client, selecting your service provider will not cause a warning message.
3. When other providers first discover your provider, they are more likely to trust your provider key if it is fetched over a commercially verified link.

The LEAP platform is designed so that it assumes you are using a commercial cert for the primary domain of your provider, but all other servers are assumed to use non-commercial certs signed by the Server CA you create.

To generate a CSR, run:

    leap cert csr

This command will generate the CSR and private key matching `provider.domain` (you can change the domain with `--domain=DOMAIN` switch). It also generates a server certificate signed with the Server CA. You should delete this certificate and replace it with a real one once it is created by your commercial CA.

The related commercial cert files are:

    files/
      certs/
        domain.org.crt    # Server certificate for domain.org, obtained by commercial CA.
        domain.org.csr    # Certificate signing request
        domain.org.key    # Private key for you certificate
        commercial_ca.crt # The CA cert obtained from the commercial CA.

The private key file is extremely sensitive and care should be taken with its provenance.

If your commercial CA has a chained CA cert, you should be OK if you just put the **last** cert in the chain into the `commercial_ca.crt` file. This only works if the other CAs in the chain have certs in the debian package `ca-certificates`, which is the case for almost all CAs.

Locations
================================

Nodes with public services can have a location specified. This allows the client to prefer to make connections to the closer nodes. This is particularly important for OpenVPN nodes.

The location stanza in a node's config file looks like this:

    {
      "location": {
        "name": "Ankara",
        "country_code": "TR",
        "timezone": "+2",
        "hemisphere": "N"
      }
    }

The fields:

* `name`: Can be anything, might be displayed to the user in the client if they choose to manually select a gateway.
* `country_code`: The [ISO 3166-1](https://en.wikipedia.org/wiki/ISO_3166-1) two letter country code.
* `timezone`: The timezone expressed as an offset from UTC (in standard time, not daylight savings). You can look up the timezone using this [handy map](http://www.timeanddate.com/time/map/).
* `hemisphere`: This should be "S" for all servers in South America, Africa, or Australia. Otherwise, this should be "N".

These location options are very imprecise, but this is good enough for our purpose. The client often does not know its own location precisely either. Instead, the client makes an educated guess at location based on the OS's timezone and locale.

If you have multiple nodes in a single location, it is best to use a tag for the location. For example:

`tags/ankara.json`:

    {
      "location": {
        "name": "Ankara",
        "country_code": "TR",
        "timezone": "+2",
        "hemisphere": "N"
      }
    }

`nodes/vpngateway.json`:

    {
      "services": "openvpn",
      "tags": ["production", "ankara"],
      "ip_address": "1.1.1.1",
      "openvpn": {
        "gateway_address": "1.1.1.2"
      }
    }
