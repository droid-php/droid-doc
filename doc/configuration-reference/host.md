# Host configuration

A Host provides Droid with the information it requires to execute commands on a
remote Host.  At minimum, Droid requires the information needed to connect to
an SSH service running on the Host.  This information can be provided in two
ways: explicitly or with a reference to a local SSH client configuration.

Explicitly providing the connection information might look like this:-

```yaml
hosts:
    myhost:
        droid_ip: "198.51.100.1"
        username: "some_user"
        keyfile: "/path/to/a/private/ssh/key"
```

Here we have given the host a label `myhost` by which we may refer to this Host
throughout the project; we have given an IP address to which Droid will
connect; we have given the name of a user account on the Host; and we have
given the path to an SSH private key with which Droid will authenticate to
the Host.

This same information might already exist in our SSH client configuration (for
example in `~/.ssh/config`) so instead of repeating it here, we may refer to a
Host in the SSH client configuration by providing a matching IP address, host
name or alias:-

```yaml
hosts:
    myhost:
        droid_ip: "198.51.100.1"
    my_other_host:
        droid_ip: "box.example.com"
    another_host:
       droid_ip: "an-alias"
```

In addition to the connection information, a Host can be configured with its
own set of variables, options to pass to the SSH client and a set of firewall
rules.

## `droid_ip`

The `droid_ip` property of a Host provides Droid with the IP address to which
Droid will connect in order to execute commands on the Host.  It may specify an
IP address, a host name or an alias understood by the SSH client.

It is not necessary to give a value for this property if either the `public_ip`
or `private_ip` properties contain an IP address because Droid will use those if
there is no value for `droid_ip`.

## `droid_port`

The `droid_port` property of a Host is the TCP port number by which Droid will
connect to the SSH service on the Host.  This is useful if the SSH service is
bound to a port other than the usual SSH port 22.

## `username`

The `username` property of a Host is the name of a user account Droid will use
to authenticate itself to the Host.  Droid will also execute commands as this
user on the Host.

## `keyfile`

The `keyfile` property of a Host is a path to a private SSH key used to
authenticate Droid to the Host.  It should be one which corresponds to a
public key known to the Host and authorised for use in SSH authentication: that
is, the corresponding public key is lodged in the `~/.ssh/authorized_keys` of a
user account on the Host.

## `public_ip`

The `public_ip` property of a Host is intended to be the publicly accessible IP
address of the Host.  Droid will connect to the public IP address in order to
execute commands on the Host when a value is not given for `droid_ip` or when
Droid is instructed to connect to `private_ip` (see the `environment` directive
in the [Project Configuration][conf-project]).

The `public_ip` property is also used in the generation of firewall rules: the
rules definitions may refer to a the `public_ip` of another Host by keyword
instead of the actual address (see the `address` property of a [Firewall
Rule][conf-fw]).

## `private_ip`

The `private_ip` property of a Host is intended to be a non-public IP address
of the Host.  Droid will connect to the private IP address in order to execute
commands on the Host when a value is given neither for `droid_ip` nor
`public_ip`.  Droid will connect to the private IP address also when Droid is
instructed to do so (see the `environment` directive in the [Project
Configuration][conf-project]).

A private IP address may be particularly useful when dealing with Hosts which
may communicate with each other over a private network, such as with many
popular Virtual Private Server providers.

The `private_ip` property is also used in the generation of firewall rules: the
rules definitions may refer to a the `private_ip` of another Host by keyword
instead of the actual address (see the `address` property of a [Firewall
Rule][conf-fw]).

## `variables`

The `variables` property of a Host is intended to provide concrete values for
the arguments of Tasks that will execute on the Host.  The values defined here
are merged with those defined elsewhere in the Project.  If two values have the
same name, then the one defined here takes precedence over those defined
elsewhere in the Project.

## `ssh_options`

The `ssh_options` property of a Host provides Droid with options to pass to the
SSH client when making an SSH connection to the Host.  The value is a mapping
of option names to their values.  For example we may wish to increase the
output from the SSH client:-

```yaml
hosts:
    myhost:
        ssh_options:
            LogLevel: "VERBOSE"
```

When making a connection to a Host, Droid automatically sets some options:-

- `IdentityFile`: set to the value given in the `keyfile` property of the Host
- `IdentitiesOnly`: set to `yes`
- `Port`: set only when the `droid_port` property of the Host is set to
  something other than 22

## `ssh_gateway`

The `ssh_gateway` property of a Host is used when the connection to the Host
should be made via an SSH Gateway.  Its value is the label given to the gateway
Host.  For example:

```yaml
hosts:
    my_gateway:
        droid_ip: "198.51.100.255"
        username: "my_gw_user"
        keyfile: "id_rsa"
    my_host:
        droid_ip: "198.51.100.1"
        username: "my_user"
        keyfile: "id_rsa"
        ssh_gateway: "my_gateway"
```

The `ssh_gateway` property is a helpful shortcut for the `ProxyCommand` SSH
option.  The example above could be achieved directly with:-

```yaml
hosts:
    my_gateway:
        droid_ip: "198.51.100.255"
        username: "my_gw_user"
        keyfile: "id_rsa"
    my_host:
        ...
        ssh_options:
            ProxyCommand: "ssh -o IdentityFile=id_rsa -o IdentitiesOnly=yes my_gw_user@198.51.100.255 nc %h %p"
```

## `firewall_policy`

The `firewall_policy` property of a Host is used by the `fw:generate` and
`fw:install` Commands in setting-up Uncomplicated Firewall (UFW) on the Host.
It is a mapping of UFW network traffic directions (incoming, outgoing, routed)
to actions (allow, deny, reject) and sets the default traffic policy for the
Host.  For example, the following policy:-

```yaml
hosts:
    my_host:
        firewall_policy:
            incoming: "deny"
            outgoing: "allow"
            routed: "reject"
```

is transformed into the following UFW commands:-

```shell
ufw default deny incoming
ufw default allow outgoing
ufw default reject routed
```

## `firewall_rules`

The `firewall_rules` property of a Host is used by the `fw:generate` and
`fw:install` Commands in setting-up Uncomplicated Firewall (UFW) on the Host.
It is a list of rules.

```yaml
hosts
    my_host:
        firewall_rules:
            - address: "all"
              port: 3306
              direction: "inbound"
              action: "deny"
```

Please see the [Firewall Rule Configuration][conf-fw] for the configuration of
firewall rules.

[conf-fw]: </configuration-reference/firewall-rule.html> "Firewall rule configuration"
[conf-module]: </configuration-reference/module.html> "Module configuration"
[conf-project]: </configuration-reference/project.html> "Project configuration"
[conf-task]: </configuration-reference/task.html> "Task configuration"
