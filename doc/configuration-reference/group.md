# Group configuration

A Group is intended to allow Hosts to be grouped into logical units, such as
"web_servers".  Tasks can be performed on all members of a Group by referring
to the Group name.

A Group also allows variables and firewall rules and policy to be applied to
all members of the Group.

## `hosts`

The `hosts` property of a Group defines the members of the Group.  It is a list
of names, each corresponding to a Host defined in the `hosts` directive of a
Droid Project (see [Project configuration][conf-project]).  For example, a set
of web servers might be defined as a Group named "web_servers":-

    groups:
        web_servers:
            hosts:
                - "web-01"
                - "web-02"
    hosts:
        web-01:
            public_ip: "198.51.100.1"
            ...
        web-02:
            ...

## `variables`

The `variables` property of a Group is intended to provide concrete values for
the arguments of Tasks.  The values defined here apply to all members of the
Group and are merged with those defined elsewhere in the Project.  Each Host
may additionally define their own variable values to augment or override those
defined here.

## `firewall_policy`

The `firewall_policy` property of a Group is used by the `fw:generate` and
`fw:install` Commands in setting-up Uncomplicated Firewall (UFW) on the members
of the Group.  It is a mapping of UFW network traffic directions (incoming,
outgoing, routed) to actions (allow, deny, reject) and sets the default traffic
policy for the Hosts in the Group.  For example, the following policy:-

    groups:
        my_group:
            firewall_policy:
                incoming: "deny"
                outgoing: "allow"
                routed: "reject"

is transformed into the following UFW commands for execution on each of the
Hosts:-

    ufw default deny incoming
    ufw default allow outgoing
    ufw default reject routed

Each Host in the Group may augment or override the policy defined here by
giving a value for the `firewall_policy` of the Host.

## `firewall_rules`

The `firewall_rules` property of a Group is used by the `fw:generate` and
`fw:install` Commands in setting-up Uncomplicated Firewall (UFW) on the members
of the Group.  It is a list of rules.

    groups
        my_group:
            firewall_rules:
                - address: "all"
                  port: 3306
                  direction: "inbound"
                  action: "deny"

Each member of the Group may define their own rules to augment those defined
here.

Please see the [Firewall rule configuration][conf-fw] for the configuration of
firewall rules.

[conf-fw]: </configuration-reference/firewall-rule.html> "Firewall rule configuration"
[conf-project]: </configuration-reference/project.html> "Project configuration"
