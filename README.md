# Terraform
Answers and/or workarounds to questions encountered when using Terraform.

## Rationale

[Terraform by Hashicorp](https://www.terraform.io/) is a great tool which allows
to automatically provision infrastracture and manage it as a code (IaC).  

The purpose of this repo is to help quickly find answers or more likely
workarounds (if not hacks) in case of bumping into the same or similar
questions. 

>> These code snippets are not solutions and although it may help to solve some
problem, it may also bring in some other issues. Use at your own risk.

## Terraform conditional expressions

The possible answer or workaround to many of answers consists of
[Terraform conditional expression](https://www.terraform.io/docs/language/expressions/conditionals.html)
(AKA the ternary operator) and
[Terraform `count` meta argument](https://www.terraform.io/docs/language/meta-arguments/count.html).  

>> When using `count` meta argument, be sure that you are ok with the 
consquences and that you do so because you know
[When to use `for_each` instead of `count`](https://www.terraform.io/docs/language/meta-arguments/count.html#when-to-use-for_each-instead-of-count).

## Terraform if else

In Terraform syntax, there is no `if else` clause. Instead, we can use
[Terraform conditional expressions](https://www.terraform.io/docs/language/expressions/conditionals.html)
as already mentioned before.  

The ternary operator can represent the answer to "`if else`" situations. So
instead of using something like:  

```
if (instance_volume_type == "ssd") {
  volume_type = "ssd";
} else {
    volume_type = "hdd";
}
```

we could rewrite that to the following form:

```
volume_type = instance_volume_type == "ssd" ? "ssd" : "hdd"
```

__Terraform how to create resource only if variable is defined?__

```
resource "openstack_compute_instance_v2" "instance" {
  count = var.instance_count != null ? var.instance_count : 0
}
```

__Terraform how to create resource only if variable has specific value?__

```
resource "openstack_lb_members_v2" "https_8000" {
  count = var.instance_name != "control_plane" ? 1 : 0
}
```

## Terraform else if

What if things get more complicated and we need to check more than one
condition? Once again we use Terraform conditional expressions give can give us
the answer. We replace a construction similar to this: 

```
if (instance_volume_type == "ssd") {
  volume_type = "ssd";
} else if (volume_type == "hdd";) {
    volume_type = "hdd";
} else {
    volume_type = "ephemeral"
}
```

with following Terraform syntax:

```
volume_type = instance_volume_type == "ssd" ? "ssd" : (
    instance_volume_type == "hdd" ? "hdd" : "ephemeral"
)
```

Although it is possible, always consider whether it is a must. If that is not
the case, better get rid of it.

__Terraform how to set variable value based on two different variables?__

```
# Associate either:
# - created floating ips if the name of the floating ip pool was defined
# - or pre-allocated floating ips if array of existing floating ips was defined
# If neither existing nor pre-allocated floating ipis are availabe, do not
# create resource

resource "openstack_compute_floatingip_associate_v2" "instance_fip" {
  count = var.instance_count

  floating_ip = var.instance_fip_pool != null ? openstack_networking_floatingip_v2.instance_fip[count.index].address : (
    var.instance_fip != null ? data.openstack_networking_floatingip_v2.instance_fip[count.index].address : null
  )
  instance_id = openstack_compute_instance_v2.instance[count.index].id
}
```

## Terraform check whether data source exists

With Terraform we can not only create and manage _resources_ but also fetch
and re-use existing _data sources_.  

Unfortunately, if the data source does not exist, Terraform will fail.  

Here, obvious question occurs: "How can we check, whether data source exists (in
order to ensure, that Terraform will not fail)?"  

The answer is simple: "We can and should not!". It is based on the fact that
[Data sources do not allow empty result without failing](https://github.com/hashicorp/terraform/issues/16380).  

This is valid when using data sources which do not have origin in the given
Terraform config files (i.e. resources which Terraform neither creates nor
manages).  

On the contrary, if the resource is created by the Terraform and used somewhere
else in the Terraform config files as a data source, it is possible to prevent
the failure by using the
[`depends_on` meta-argument](https://www.terraform.io/docs/language/meta-arguments/depends_on.html).  

## Terraform resource create timeout

Sometimes, the resource creation can take a __lot__ of time.  

One good example is an instance which boots from volume which is created from a
large image.  

First, the volume is created. But because the image from which the instance is
created is quite a size, it takes more than five minutes to create such volume.
And so, the instance resource fails due to its creation timeout.  

We could work around this and use the
[`depends_on` meta-argument](https://www.terraform.io/docs/language/meta-arguments/depends_on.html)
in the instance resource block, so that the instance would not be created before
the volume is created.  

And so I tried. Guess what happend? Yes, you're right! Now the volume resource
(when `count` set to more than 1) failed because of the creation timeout. So now
what?  

Soon I found out, that Terraform has an answer to that which is "obviously"
called [Operation timeouts](https://www.terraform.io/docs/language/resources/syntax.html#operation-timeouts).  

Terraform docs state that only some resources support `timeouts` block. I was
working with Terraform OpenStack provider. Alas, no trace of timeouts support in
its docs at all. But I've tried it anyway. __And it worked__. So, Terraform 
`timeouts` block can save the day, even if it has not documented yet.

__Terraform timeouts block works, even if not documented in providers resources__

```
resource "openstack_blockstorage_volume_v2" "volume" {
  count       = var.instance_count

  timeouts {
    create = "60m"
    update = "10m"
    delete = "1h"
  }
}
```

## How to contribute

Thank you for reading so far. Did/do you happen to:
- find any error or typo?
- have a more elegant answer, workaround or better solution?
- have any other Terraform questions?

If so, you are welcome to open pull request.
