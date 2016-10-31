# Firewall rule configuration

The Droid Commands `fw:generate` and `fw:install` use Firewall rule
configuration of a Host to set-up Uncomplicated Firewall (UFW) on the Host.
Each rule is transformed into the arguments needed to execute the `ufw` program
on the Host.  For example, the following rule:-

    - address: "198.51.100.60"
      port: 3306
      direction: "outbound"
      action: "allow"
      comment: "Allow MySQL connections to the db server (198.51.100.60)"

is transformed into the following command:-

    ufw allow out proto tcp from any to 198.51.100.60 port 3306

An example of a rule for incoming traffic:-

    - address: "198.51.100.1"
      port: 22
      direction: "inbound"
      action: "allow"
      comment: "Allow SSH connections from Droid (198.51.100.1)"

which is transformed into the following command:-

    ufw allow in proto tcp from 198.51.100.1 to any port 22

It is important to note the `fw:install` command will activate the UFW rules
immediately.  The standard firewall policy employed by the `fw:install` command
is to deny all incoming traffic by default.  Thus there is the risk of locking
oneself out of a Host unless its `firewall_policy` (see [Host
configuration][conf-host]) is to allow incoming traffic or there is a specific
Rule to allow incoming traffic to the SSH service.  The previous example is of
such a rule: it allows Droid running on a specific machine to reach the SSH
service of the Host.

## `address`

The `address` property of a Rule identifies a remote host, from the perspective
of the Host to which the Rule applies.  A value is always required.

A value of "all" is interpreted as "any remote host".  For example to allow all
incoming traffic to a web service:-

    - address: "all"
      port:443
      direction: "inbound"
      action: "allow"
      comment: "Allow incoming HTTPS traffic from anywhere"
    - address: "all"
      port: 80
      direction: "inbound"
      action: "allow"
      comment: "Allow incoming HTTP traffic from anywhere"

The value of the `address` property may also be a specific IP address or it may
be the label of another Host as defined in the `hosts` directive of a Droid
Project (see [Project configuration][conf-project]):-

    hosts:
        web-01:
            public_ip: "198.51.100.1"
            ...
        database-01:
            ...
            firewall_rules:
                - address: "web-01"
                  port: 3306
                  ...
                  comment: "Allow incoming MySQL traffic from the web server."

In the above example, the Rule for the Host labelled `database-01` uses the
address `web-01` which is interpreted as the public IP address of that Host.  A
rule can specify the private IP of a Host by appending `:private` to the
address label:-

    hosts:
        web-01:
            private_ip: "192.0.2.1"
            ...
        database-01:
            ...
            firewall_rules:
                - address: "web-01:private"
                  ...

## `port`

The `port` property of a Rule is a TCP or UDP port number corresponding to the
service to which traffic is destined.  A value is always required.

## `protocol`

The `protocol` property of a Rule is either `tcp` or `udp`, corresponding
respectively to the TCP and UDP Internet protocols.  The default value, when
one is not given, is `tcp`.

## `direction`

The `direction` property of a Rule specifies whether the traffic being
described is incoming or outgoing, from the perspective of the Host.  The value
is either `inbound` or `outbound`.  The default value, when one is not given,
is `inbound`.

## `action`

The `action` property of a Rule determines the action of the firewall when
traffic matches the Rule.  The value may be one of `allow`, `reject` or `deny`.
The default value, when one is not given, is `allow`.

Traffic is allowed to pass unhindered when the value is `allow`.

Traffic is rejected at the initial connection attempt when the value is
`reject`.  That is, a TCP connection attempt is reset; a UDP datagram elicits
an ICMP Port Unreachable message.

Traffic is silently discarded when the value is `deny`.

## `comment`

The `comment` property of a Rule is intended to convey a short amount of human
readable information about the rule, such as a description of or reason for the
Rule.

[conf-host]: </configuration-reference/host.html> "Host configuration"
[conf-project]: </configuration-reference/project.html> "Project configuration"
