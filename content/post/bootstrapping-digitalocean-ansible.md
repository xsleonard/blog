+++
title = "Bootstrap the Ansible user from root"
tags = ["ansible"]
date = "2020-01-06T00:00:00+00:00"
+++


When configuring a new server with Ansible, I wanted to create a new user, then switch to that new user for the remainder of the playbook. As with any Ansible playbook, it should be idempotent. The way to do accomplish this was not straightforward, due to the way the Ansible SSH user behaves.

### The playbook layout

{{< gist xsleonard b1c1cd5e46cbd3fb13ea559eec6068f7 "00-project-layout.txt" >}}

There are two roles: `maybecreateuser` and `createuser`.

The `maybecreateuser` role wraps the `createuser` with logic to test if the user exists and to handle the variable setup required to swap `ansible_user`. The `createuser` role contains the user creation tasks.

### Testing if the user exists

{{< gist xsleonard b1c1cd5e46cbd3fb13ea559eec6068f7 "01-maybecreateuser-main.yml" >}}

Key points:

* Stash the `ansible_user` into another variable, before swapping in the value of `initial_user`. `initial_user` is a variable defined by this role and should be the name of the initial root user on the server.
* Test if the user already exists by trying to SSH explicitly

### Creating the user

{{< gist xsleonard b1c1cd5e46cbd3fb13ea559eec6068f7 "02-createuser-main.yml" >}}

* Create a user with no password set, to avoid dealing with password handling
* Copy the `~/.ssh/authorized_keys` file from `initial_user`, in order to SSH in as this new user. This assumes your `initial_user` has an `authorized_keys` file already (this is true on DigitalOcean if you provide your pubkey before creating the server).
* Give the user sudo rights without a password

None of these are hard requirements. You may want your user to have a password and not give it `nopasswd` sudo. You may want to use a different SSH key.

### Using it in a playbook

{{< gist xsleonard b1c1cd5e46cbd3fb13ea559eec6068f7 "04-site.yml" >}}

When using the `maybecreateuser` role, `gather_facts: no` must be set in the playbook. Otherwise, ansible will try to do `gather_facts` which requires an SSH connection, but the `ansible_user` might not exist yet.

### Configuring inventory

{{< gist xsleonard b1c1cd5e46cbd3fb13ea559eec6068f7 "05-inventory" >}}

Pass the `initial_user` if necessary to configure hosts whose initial user is not `root`.

Set the `ansible_user` to whatever you want, or don't set it. 
