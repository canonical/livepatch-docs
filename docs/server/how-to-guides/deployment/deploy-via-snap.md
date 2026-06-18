---
myst:
  html_meta:
    description: "How to deploy via snap with Livepatch on-premises."
---


(server-how-to-guides-getting-started-with-the-livepatch-server-snap)=

# How to deploy via Snap

The Canonical Livepatch Server enables the delivery of live kernel patches to Livepatch clients, allowing reboots of critical infrastructure to be scheduled at a convenient time.

This guide covers setting up the Livepatch Server snap.

> The server snap is not designed for high-availability setups.

## Prerequisites

At minimum, the server requires a PostgreSQL instance (at least version 12) to persist data. For simplicity, this guide uses Docker to illustrate the server setup. An existing PostgreSQL instance can also be used if one is available.

> For a production environment, follow the [PostgreSQL installation tutorial](https://ubuntu.com/server/docs/databases-postgresql) to install PostgreSQL with persistent storage.

To start a PostgreSQL instance in Docker, run:

```
docker run \
  --name postgresql \
  -e POSTGRES_USER=livepatch \
  -e POSTGRES_PASSWORD=testing \
  -p 5432:5432 \
  -d postgres:12.11
```

## Install the snap

To install the server snap, run:

```
sudo snap install canonical-livepatch-server
```

## Migrate the database

The snap includes an internal tool for migrating a PostgreSQL database with the Livepatch server schema. To migrate the database, run:

```
canonical-livepatch-server.schema-tool \
  postgresql://livepatch:testing@localhost:5432/livepatch
```

## Configure the database connection

All configuration for the Livepatch server snap is handled within the snap daemon. To update Livepatch to target the DSN of the PostgreSQL instance, run:

```
sudo snap set canonical-livepatch-server \
  lp.database.connection-string=postgresql://livepatch:testing@localhost:5432/livepatch
```

## Verify the server is running

To check the server is running successfully, run:

```
sudo snap logs canonical-livepatch-server.livepatch -n 100
```

## Obtain an Ubuntu Pro token

Customers of Ubuntu Pro with access to Livepatch on-premises can enable on-premises mode within the snap, in the same way as for the charm.

The Ubuntu Pro token is available from: https://ubuntu.com/pro/dashboard

## Set the Ubuntu Pro token

The server configuration can be updated through the snap daemon. To update the server to use the token and enable Livepatch on-premises, run:

```
sudo snap set canonical-livepatch-server token=<Ubuntu Pro token>
```

## Install the administration tool

To manage Livepatch, an administrator tool is available as a snap. Install it with:

```
sudo snap install canonical-livepatch-server-admin
```

The administrator tool needs to know where the Livepatch server is hosted. In an all-in-one setup on a single machine, this is `http://localhost:8080`. Export an environment variable as follows:

```
export LIVEPATCH_URL=http://localhost:8080
```

## Set up authentication

For the administrator tool to log in to the server, basic authentication must be set up in the snap daemon. For the purpose of this guide, the username is `admin` and the password is `admin123`.

> Special characters are escaped using single quotes. The password must be bcrypt hashed.

```
sudo snap set canonical-livepatch-server \
  lp.auth.basic.users='admin:$2y$10$c25NVkdeIMqWdbgR4883YuE/s2CT1mCmGPm5Ma1XbUqGqM26ClTGe'
```

To generate a custom password hash, run:

```
sudo apt-get install apache2-utils
htpasswd -bnBC 10 <username> <password>
```

Next, enable basic authentication:

```
sudo snap set canonical-livepatch-server lp.auth.basic.enabled=true
```

Finally, log in with the administrator tool, assuming the example username and password have been used:

```
canonical-livepatch-server-admin.livepatch-admin login -a admin:admin123
```

## Synchronize with hosted Livepatch

Once the on-premises Livepatch server is running and fully configured, synchronize patches from hosted Livepatch into the server. Run the following within the administrator tool:

```
canonical-livepatch-server-admin.livepatch-admin sync trigger
```

To set the server to automatically sync patches from Canonical's servers every 12 hours, run:

```
sudo snap set canonical-livepatch-server lp.patch-sync.enabled=true
sudo snap set canonical-livepatch-server lp.patch-sync.interval=12h
```

The default configuration stores patches on the local filesystem of the server. The patches can be viewed at:

```
ls /var/snap/canonical-livepatch-server/common/patches/
```

## Expose the server

By default the snap listens on `localhost:8080` and is not accessible to requests from external networks. To check the current setting, run:

```
sudo snap get canonical-livepatch-server lp.server.server-address
localhost:8080
```

To listen for all incoming connections on a different port, run:

```
sudo snap set canonical-livepatch-server lp.server.server-address=0.0.0.0:<port>
```

Update the `LIVEPATCH_URL` environment variable to ensure the admin tool can still access the server. To access the server from a remote machine, replace `localhost` with the IP address of the machine running the Livepatch server:

```
export LIVEPATCH_URL=http://localhost:<port>
```

## Next steps

The on-premises Livepatch server is now configured to synchronize with hosted Livepatch.

For further reading, consult the how-to guides.
