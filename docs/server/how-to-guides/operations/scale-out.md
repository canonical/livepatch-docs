---
myst:
  html_meta:
    description: "How to scale out with Livepatch on-prem."
---

(server-how-to-guides-how-to-scale-out-a-livepatch-on-prem-deployment)=

# Scale out a Livepatch on-premises deployment

Scaling out a Livepatch deployment may be necessary in the following scenarios:

1. High traffic due to a large number of machines being serviced
2. High availability setups

The Livepatch on-premises deployment consists of three main components: `haproxy` reverse-proxying requests, `postgresql` storing Livepatch data, and the `Livepatch Server` itself. Any or all of these components can be scaled out with additional units by running the `juju` command with the following syntax:

```
juju add-unit <component> -n <number of units>
```

Where `<component>` is `haproxy`, `postgresql` or `livepatch` and `<number of units>` is the number of additional units required.

![image5|622x350](/_static/images/ydBPKHEv0bSYdpJZHTy8pjUDGps.png)

## Scale out haproxy

To deploy more than one ingress haproxy unit, run the following Juju command:

```
juju add-unit haproxy -n 1
```

If more than one haproxy unit is in use, the DNS entry pointing to the Livepatch Server should contain links to all the haproxy units.

## Scale out postgresql

Scaling out postgresql deploys additional follower units that can take over if the leader fails.

To deploy additional postgresql units, run the following Juju command:

```
juju add-unit postgrseql -n 1
```

## Scale out Livepatch

Deploying additional Livepatch units allows them to share the load of handling machine requests. The filesystem patch storage type is not compatible with a scaled-out Livepatch setup.

To deploy additional Livepatch units, run the following Juju command:

```
juju add-unit livepatch -n 1
```

## Get new resource tokens

After the units have been added, run the following command for each new unit:

```
juju run-action livepatch/{i} get-resource-token
```

Replace `{i}` with the number of each new Livepatch unit from the `juju status` output.