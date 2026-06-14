---
myst:
  html_meta:
    description: "How to enable client on ubuntu core with Livepatch client."
---


(client-how-to-guides-how-to-enable-the-livepatch-client-on-ubuntu-core)=

# How to enable the Livepatch client on Ubuntu Core

Canonical Livepatch supports live kernel patching on Ubuntu Core machines. Livepatch is supported on Core20+ for amd64 and Core26+ for arm64 architectures. Canonical Livepatch is typically enabled using the Pro Client on classic Ubuntu machines. The Pro Client is not available for install on Ubuntu Core machines, therefore users must enable Livepatch client directly. To do so, follow these steps on the Ubuntu Core machine:

1. Install canonical-livepatch:

```shell
sudo snap install canonical-livepatch
```

2. Install jq and curl

```shell
sudo snap install jq curl
```

3. Obtain a contract resource token using the Ubuntu Pro token from [ubuntu.com/pro](http://ubuntu.com/pro) dashboard.

```shell
export pro_token=<pro_token>

body="{\"architecture\":\"$(uname -m)\", \"hostType\":\"physical\", \"machineId\":\"$(cat /etc/machine-id)\", \"os\":{\"distribution\":\"$(. /etc/os-release && echo $PRETTY_NAME)\", \"kernel\":\"$(uname -r)\", \"release\":\"$(. /etc/os-release && echo $VERSION_ID)\", \"series\":\"core$(. /etc/os-release && echo $VERSION_ID)\", \"type\":\"Linux\"}}" && curl -X POST -H "Authorization: Bearer $pro_token" -H "Content-Type: application/json" https://contracts.canonical.com/v1/context/machines/token -d "$body" | jq '.resourceTokens | map(select(.type=="livepatch"))'

```

**Note:** Replace <pro_token> with your Ubuntu Pro token obtained for [ubuntu.com/pro](http://ubuntu.com/pro) dashboard.

You should see the following output:

```json
[
    {
        "token": "<resource_token>",
        "type": "livepatch"
    }
]
```

4. Enable Livepatch client using the \<resource_token> from the previous step.

```
sudo canonical-livepatch enable <resource_token>
```

At this point, the Canonical Livepatch client has been enabled on your Ubuntu Core machine.
