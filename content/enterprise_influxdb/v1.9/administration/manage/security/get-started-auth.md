---
title: Get started with authentication and authorization
description: >
  Learn the basics of managing authentication and authorization in InfluxDB Enterprise
menu:
  enterprise_influxdb_1_9:
    name: Get started with authentication and authorization
    parent: Manage security
weight: 30
related:
  - /enterprise_influxdb/v1.9/guides/fine-grained-authorization/
  - /{{< latest "chronograf" >}}/administration/managing-influxdb-users/
---

To secure and manage access to an InfluxDB Enterprise cluster, consider the following two aspects:
- *authentication* (verifying a user's identity)
- *authorization* (verifying what the user has access to)

## Enable authentication

Authentication is disabled by default in InfluxDB and InfluxDB Enterprise.
Once you have a working cluster
(that is, after [installing the data nodes](/enterprise_influxdb/v1.9/introduction/install-and-deploy/installation/data_node_installation/)),
we recommend enabling authentication right away to control access to you cluster.

Authentication must be enabled _**before**_ authorization can be managed.

To enable authentication in a cluster, do the following:

1. Set `auth-enabled` to `true` in the `[http]` section of the config files for all meta and data nodes:
   ```toml
   [http]
     # ...
     auth-enabled = true
   ```
1. Next, create an admin user (if you haven't already).
   Using the [`influx` CLI](/enterprise_influxdb/v1.9/tools/influx-cli/),
   run the following command:
   ```
   CREATE USER admin WITH PASSWORD 'mypassword' WITH ALL PRIVILEGES
   ```
1. Restart InfluxDB Enterprise.
   Once restarted, InfluxDB Enterprise checks user credentials on every request
   and only processes requests with valid credentials.

## Permissions in InfluxDB Enterprise

Compared to InfluxDB OSS, InfluxDB Enterprise has an expanded set of permissions,
In InfluxB 1.x OSS, there only database-level privileges: `READ` and `WRITE`.
A third permission, `ALL`, grants admin privileges.

InfluxDB Enterprise has these permissions and more.
In addition to these three basic database-level permissions,
_which can only be granted by InfluxQL_,
InfluxDB Enterprise's [full set of permissions](/enterprise_influxdb/v1.9/administration/manage/security/authentication_and_authorization-api/#list-of-available-privileges)
allows for controling read and write access to data for all databases at once,
as well as cluster-management actions like creating or deleting resources.

## Manage user authorization

There are three ways to manage authorizations in InfluxDB Enterprise.
Each is useful in different scenarios.
They are:

- the [`influx` CLI](#influxql) (1.x) with [InfluxQL](#manage-read-and-write-privileges-with-influxql)
- [Chronograf](#manage-specific-privileges-with-chronograf)
- the [InfluxDB Enterprise meta API](#influxdb-enterprise-meta-api)

Each of these allows you to manage permissions for specific users.

### Manage read and write privileges with InfluxQL

{{% note %}}
InfluxQL provides a quick way to grant read and write privileges.
However, you can only grant `READ`, `WRITE`, and `ALL PRIVILEGES` privileges with this method.
To apply more fine-grained permissions, see [Manage specific privileges with Chronograf](#manage-specific-privileges-with-chronograf).
{{% /note %}}

InfluxQL can be used to manage basic read and write privileges.
For example, you can grant Alice the ability to write to a database *X*,
and then grant Bob the ability to read from that database.

To demonstrate, let's create a new user and allow her some access.
First, log in as an admin user.
The simplest way to do this is to use the `auth` command.

```sh
$ influx
Connected to http://localhost:8086 version 1.9.5-c1.9.5
InfluxDB shell version: 1.8.7
> auth
```

This will prompt for username and password.
Enter the credentials for the admin you created above in ["Enable authentication"](#enable-authentication).

Then run:

```sh
> CREATE USER alice WITH PASSWORD 'admin'
```

This creates a regular, non-admin user.

Since we are logged in as an admin user, we can create a database.
Create a database called `test` by running

```sh
> CREATE DATABASE test
```

Now we can grant Alice permission to write to the `test` database.

```sh
> GRANT WRITE ON test TO alice
```

We can inspect Alice's permissions with

```sh
> SHOW GRANTS FOR alice
database privilege
-------- ---------
test     WRITE
```

### Manage specific privileges with Chronograf

The Chronograf user interface can manage the [full set of permissions](/enterprise_influxdb/v1.9/administration/manage/security/authentication_and_authorization-api/#list-of-available-privileges).

The permissions listed in Chronograf (and available through the API) are global for the cluster.
Outside of [FGA](), the only database-level permissions available are the basic `READ` and `WRITE`.
These can only be managed using [the `influx` CLI and InfluxQL](#manage-read-and-write-privileges-with-influxql).

To manage privileges in Chronograf:

1. Login to Chronograf as an admin user.
2. Click on **InfluxDB Admin** → **Users**.
3. Each user has a dropdown for **Permissions**.
   Select the desired permissions and click **Apply**.

### InfluxDB Enterprise meta API

Programmatically manage permissions with the InfluxDB Enterprise API.

```sh
./influx
Connected to http://localhost:8086 version 1.9.5-c1.9.5
InfluxDB shell version: unknown

> create user admin with password 'pass' with all privileges
> create user bob with password 'pass'
> grant read on mydb to bob
```

then turn on admin perms

```sh
./influx -username bob -password pass
Connected to http://localhost:8086 version 1.9.5-c1.9.5
InfluxDB shell version: unknown
> use mydb
Using database mydb
> delete where 1=1
ERR: error authorizing query: bob not authorized to execute statement 'DELETE WHERE 1 = 1'
```



```
$ curl -Lv http://localhost:8091/user\?name\=bob | jq
{
  "users": [
    {
      "name": "bob",
      "hash": "$2a$10$rnR2Q/8b4GXij1YWLVQMfO3l7B.tXbv9Hp3jNNy2RNHXuH/r83oQa",
      "permissions": {
        "mydb": [
          "ReadData"
        ]
      }
    }
  ]
}
```

```
$ curl -Lv http://localhost:8091/user -d '{"action":"add-permissions","user":{ "name": "bob", "permissions": {"mydb": ["DropData"]}}}'
```

```
curl http://localhost:8091/user | jq
{
  "users": [
    {
      "name": "admin",
      "hash": "$2a$10$LZVXUAZcN2OCzmTcnuoQLubuRRrBOlNcHF4NMwXPGFHA1mEfz3oeS",
      "permissions": {
        "": [
          "ViewAdmin",
          "ViewChronograf",
          "CreateDatabase",
          "CreateUserAndRole",
          "DropDatabase",
          "DropData",
          "ReadData",
          "WriteData",
          "ManageShard",
          "ManageContinuousQuery",
          "ManageQuery",
          "ManageSubscription",
          "Monitor"
        ]
      }
    },
    {
      "name": "bob",
      "hash": "$2a$10$rnR2Q/8b4GXij1YWLVQMfO3l7B.tXbv9Hp3jNNy2RNHXuH/r83oQa",
      "permissions": {
        "mydb": [
          "DropData",
          "ReadData"
        ]
      }
    }
  ]
}
```

Now bob is authorized to do delete on mydb

```
> delete where 1=1
>
```

For more information on using the meta API,
see [here](/enterprise_influxdb/v1.9/administration/manage/security/authentication_and_authorization-api).
