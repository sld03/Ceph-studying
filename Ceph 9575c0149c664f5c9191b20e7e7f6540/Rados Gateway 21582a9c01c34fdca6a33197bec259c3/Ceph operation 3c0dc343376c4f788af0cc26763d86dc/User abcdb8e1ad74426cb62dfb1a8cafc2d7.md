# User

# 1. User

![Untitled](User%20abcdb8e1ad74426cb62dfb1a8cafc2d7/Untitled.png)

There are 2 types o Users in Ceph

- **User:** The term “user” refers to user of the S3 interface.
- **Subuser:** The term “subuser” refers to a user of the Swift interface. A subuser is associated with a user.

Optionally, the users can belong to [Accounts](https://docs.ceph.com/en/latest/radosgw/account/) for ease of management.

# 2. User mamagement

Create an User

```bash
radosgw-admin user create --uid={username} --display-name="{display-name}" [--email={email}]
```

![Untitled](User%20abcdb8e1ad74426cb62dfb1a8cafc2d7/Untitled%201.png)

Here we get the key for this user

```bash
 {
    "user": "sld",
    "access_key": "EDD2BVX1LWAW3SBI43RB",
    "secret_key": "yaSaNC7Y8QLBloIZorNfQu05PpKd5223K0Ngar55"
}
```

### Get User Info

```bash
radosgw-admin user info --uid=johndoe
```

![Untitled](User%20abcdb8e1ad74426cb62dfb1a8cafc2d7/Untitled%202.png)

### Modify user info

```bash
radosgw-admin user modify --uid=johndoe --display-name="John E. Doe"
```

### Suspend and enable user

```bash
radosgw-admin user suspend --uid=johndoe
radosgw-admin user enable --uid=johndoe
```

### Remove a user

```bash
radosgw-admin user rm --uid=johndoe
```

Use option `—pruge-data` to purges all data associated with the UID, 

`—purge-key` to purges all key associated with the UID

### Add or remove admin capability

```bash
radosgw-admin caps add --uid={uid} --caps={caps}
# and to remove
radosgw-admin caps rm --uid=johndoe --caps={caps}
```

Example

```bash
radosgw-admin caps add --uid=johndoe --caps="users=*;buckets=*"
```

Capability option

```bash
--caps="[users|buckets|metadata|usage|zone|amz-cache|info|bilog|mdlog|datalog|user-policy|oidc-provider|roles|ratelimit|user-info-without-keys]=[\*|read|write|read, write]"
```