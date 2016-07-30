---
title: "Swift"
description: "Swift"
date: "2014-04-26"
---

<i class="fa fa-space-shuttle"></i>Swift
----------------------------------------

Swift refers to [Openstack Object Storage](http://www.openstack.org/software/openstack-storage/).
Commercial implementations of that being:

  * [Rackspace Cloud Files](http://www.rackspace.com/cloud/files/)
  * [Memset Memstore](http://www.memset.com/cloud/storage/)

Paths are specified as `remote:container` (or `remote:` for the `lsd`
command.)  You may put subdirectories in too, eg `remote:container/path/to/dir`.

Here is an example of making a swift configuration.  First run

    rclone config

This will guide you through an interactive setup process.

```
No remotes found - make a new one
n) New remote
s) Set configuration password
n/s> n
name> remote
Type of storage to configure.
Choose a number from below, or type in your own value
 1 / Amazon Drive
   \ "amazon cloud drive"
 2 / Amazon S3 (also Dreamhost, Ceph)
   \ "s3"
 3 / Backblaze B2
   \ "b2"
 4 / Dropbox
   \ "dropbox"
 5 / Google Cloud Storage (this is not Google Drive)
   \ "google cloud storage"
 6 / Google Drive
   \ "drive"
 7 / Hubic
   \ "hubic"
 8 / Local Disk
   \ "local"
 9 / Microsoft OneDrive
   \ "onedrive"
10 / Openstack Swift (Rackspace Cloud Files, Memset Memstore, OVH)
   \ "swift"
11 / Yandex Disk
   \ "yandex"
Storage> 10
User name to log in.
user> user_name
API key or password.
key> password_or_api_key
Authentication URL for server.
Choose a number from below, or type in your own value
 1 / Rackspace US
   \ "https://auth.api.rackspacecloud.com/v1.0"
 2 / Rackspace UK
   \ "https://lon.auth.api.rackspacecloud.com/v1.0"
 3 / Rackspace v2
   \ "https://identity.api.rackspacecloud.com/v2.0"
 4 / Memset Memstore UK
   \ "https://auth.storage.memset.com/v1.0"
 5 / Memset Memstore UK v2
   \ "https://auth.storage.memset.com/v2.0"
 6 / OVH
   \ "https://auth.cloud.ovh.net/v2.0"
auth> 1
User domain - optional (v3 auth)
domain> Default
Tenant name - optional
tenant> 
Tenant domain - optional (v3 auth)
tenant_domain>
Region name - optional
region> 
Storage URL - optional
storage_url> 
Remote config
AuthVersion - optional - set to (1,2,3) if your auth URL has no version
auth_version> 
--------------------
[remote]
user = user_name
key = password_or_api_key
auth = https://auth.api.rackspacecloud.com/v1.0
tenant = 
region = 
storage_url = 
--------------------
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d> y
```

This remote is called `remote` and can now be used like this

See all containers

    rclone lsd remote:

Make a new container

    rclone mkdir remote:container

List the contents of a container

    rclone ls remote:container

Sync `/home/local/directory` to the remote container, deleting any
excess files in the container.

    rclone sync /home/local/directory remote:container

### Specific options ###

Here are the command line options specific to this cloud storage
system.

#### --swift-chunk-size=SIZE ####

Above this size files will be chunked into a _segments container.  The
default for this is 5GB which is its maximum value.
      
### Modified time ###

The modified time is stored as metadata on the object as
`X-Object-Meta-Mtime` as floating point since the epoch accurate to 1
ns.

This is a defacto standard (used in the official python-swiftclient
amongst others) for storing the modification time for an object.

### Limitations ###

The Swift API doesn't return a correct MD5SUM for segmented files
(Dynamic or Static Large Objects) so rclone won't check or use the
MD5SUM for these.

### Troubleshooting ###

#### Rclone gives Failed to create file system for "remote:": Bad Request ####

Due to an oddity of the underlying swift library, it gives a "Bad
Request" error rather than a more sensible error when the
authentication fails for Swift.

So this most likely means your username / password is wrong.  You can
investigate further with the `--dump-bodies` flag.

#### Rclone gives Failed to create file system: Response didn't have storage storage url and auth token ####

This is most likely caused by forgetting to specify your tenant when
setting up a swift remote.
