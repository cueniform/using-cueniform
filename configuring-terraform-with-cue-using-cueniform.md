# Configuring Terraform with CUE using Cueniform

## Quick start

Here are some steps to get you started:

1. [Install CUE](https://cuelang.org/docs/install/)
   version `v0.6.0-alpha.1` or later

1. Initialise your local CUE module

   ```shell
   # shell: cut & paste
   cue mod init terraform.test
   ```

   The module name (`terraform.test`) is totally up to you, and is private.
   It's needed for CUE to distiguish "your" CUE content from 3rd party content
   (i.e. Cueniform!). You can select anything you like - `terraform.test` is as
   good as any choice, initially. Changing this name later is trivial.

1. Clone some Cueniform repos

   ```shell
   # shell: cut & paste
   git clone \
       https://github.com/cueniform/terraform-core-schema \
       cue.mod/pkg/cueniform.com/x/terraform-core
   git clone \
       https://github.com/cueniform/cueniform \
       cue.mod/pkg/cueniform.com/x/cueniform
   ```

   Cueniform keeps the core schemata required by Terraform separate from
   Cueniform itself.

1. Provide a place where Cueniform's configuration can live, away from your
   Terraform configuration.

   ```shell
   # shell: cut & paste
   mkdir -p cnifm
   ```

   Ultimately, everything in the `cnifm` directory will be managed by Cueniform
   itself, but initially you'll have to help it along ... :-)

1. Teach Cueniform about your Terraform version, and some other low-level
   detail:

   ```shell
   # shell: cut & paste
   cat <<HERE >cnifm/setup.cue; cue fmt ./cnifm
   package cnifm
   import (
     "cueniform.com/x/cueniform"
     "cueniform.com/x/terraform-core/1.4/core"
   )
   X=#Validate: cueniform.#Settings & {
     #Terraform:    core
     #Entities:     entities
     configuration: X.#Configuration
   }
   HERE
   ```

   This file will be managed by Cueniform in the future, but **you *don't* need
   to understand** what's going on here right now!

1. Teach Cueniform about the currently-empty set of entities your Terraform
   configuration will be using:

   ```shell
   # shell: cut & paste
   cat <<HERE >cnifm/entities.cue; cue fmt ./cnifm
   package cnifm
   entities: {}
   HERE
   ```

   The `entities` namespace is where we'll be referencing all the upstream
   provider schemata published by Cueniform in the future.
   Right now you'll need to generate *manual* schemata for all the resource and
   data-source types you use. Don't worry: a default, opinion-less schema is
   trivial to generate in CUE, so you only need to dive into provider
   documentation on the Terraform Registry *if you want to*! We'll see
   [how to do this](#teaching-cueniform-about-new-resources-and-data-sources)
   shortly ...

1. Start configuring Terraform

   ```shell
   # shell: cut & paste
   cat <<HERE >terraform.cue; cue fmt
   package terraform
   import "terraform.test/cnifm"
   cnifm.#Validate

   configuration: {
     output: hello: {
       value: "Hello from Cueniform!"
     }
   }
   HERE
   ```

1. Export your config into Terraform JSON

   ```shell
   # shell: cut & paste
   cue export -e cueniform -f -o config.tf.json
   ```

1. Run Terraform

   ```shell
   # shell: cut & paste
   terraform apply -auto-approve
   ```

1. Version control your important files:

   - `cnifm/setup.cue`
   - `cnifm/entities.cue`
   - `terraform.cue`
   - `config.tf.json`
   - the contents of `cue.mod`
     - you might prefer to use a git-submodule rather than commiting the
       directory
     - CUE and Cueniform don't care how you deal with this!

## Teaching Cueniform about new resources and data-sources

By including `cnifm.#Validate` in your `terraform.cue` file, you're including a
nested set of CUE files and assertions that try and ensure that each and every
resource and data-source is validated against a matching schema inside your
`entities` namespace.

But right now, your `entities` namespace (in `cnifm/entities.cue`) is *empty*.

Populating and expanding that `entities` namespace *automatically* is part of
the work that we're doing with Cueniform, behind the scenes. But until that's
ready, you need to teach Cueniform about the schemata of the resources and
data-sources you're using manually.

But don't worry - **it's not complicated!** Doing this is really, *really*
easy, and as an added benefit you get a place to put policy assertions later.

Let's imagine we're going to start using Cueniform to teach Terraform to manage
GitHub repositories with the `github_repository` resource type. Its
documentation is
[here](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/repository),
but **you don't need to read it**.

Ultimately, translating the information from that documentation into CUE
schemata will be Cueniform's responsibility but, right now, we're going to
cheat: we're going to teach CUE that *anything* is allowed, and to give
ourselves space to add assertions later on, as they become important to us.

Let's create a new file and add our custom, empty schema for
`github_repository` resources:

```shell
# shell: cut & paste
cat <<HERE >cnifm/entities.cue; cue fmt ./cnifm
package cnifm
entities: {
  github_repository: #Resource: {...}
}
HERE
```

That `...` is CUE's way of saying "any fields are permitted here".

Now we can add `github_repository` entries in `terraform.cue`, and CUE will be
happy:

```shell
# shell: cut & paste
cat <<HERE >>terraform.cue; cue fmt
configuration: resource: {
  github_repository: {
    "my.first.repo": {
      name: "my.first.repo"
    }
  }
}
HERE

cue export -e cueniform -f -o config.tf.json
```

But ... **what's the *point* of adding a schema** that doesn't impose any
opinions?

Well, first of all, it's implicitly checking our resource type spelling! If we
mistype the `github_repository` resource type anywhere (e.g. as `github_repo`),
then CUE will tell us.

But more importantly, let's imagine that at some point in the future, we make a
mistake in our configuration: we create a repo that we thought was going to be
private, but we forgot that the `github` provider's silent default is to create
**public** repos.

After we've realised what we've done wrong and we've fixed it, we want to add a
schema assertion to prevent us from ever making this mistake again. We want to
make sure that every repo explicitly states if it's public or private. The
`cnifm/entities.cue` file, in the `entities` namespace, is the right place to
put that assertion:

```shell
# shell: cut & paste
cat <<HERE >cnifm/entities.cue; cue fmt ./cnifm
package cnifm
entities: {
  github_repository: #Resource: {
    visibility!: "public" | "private"
    ...
  }
}
HERE
```

Now all our `github_repository` resources have to state *explicitly* if the
repos they create are "public" or "private", without relying on the `github`
provider's implicit default.

Let's see that in action:

```shell
# shell: cut & paste
cue export -e cueniform -f -o config.tf.json
```

You should see CUE telling you something similar to this:

```text
cueniform.resource.github_repository.my_first_repo.visibility:
  field is required but not present
```

Let's fix that:

```shell
# shell: cut & paste
cat <<HERE >>terraform.cue; cue fmt
configuration: resource: github_repository: "my.first.repo": {
  visibility: "public"
}
HERE
cue export -e cueniform -f -o config.tf.json
cat config.tf.json
```

If we were to run `terraform apply` now, it would try (and fail!) to create a
GitHub repository. Instead of doing that, the above shell snippet just shows
the `config.tf.json` file's contents, letting you see that it includes the
`visibility` field.

CUE lets us build up these schema assertions bit by bit, over time: putting
effort in where it brings us value, and being lazy where *that*'s valuable :-)

Notice that in `terraform.cue` CUE also *literally* lets us build up our
data, piece by piece. It unifies all structs together at runtime, as CUE
[doesn't care about the order in which you define your data](https://cuelang.org/docs/tutorials/tour/intro/order/)

Ultimately, Cueniform will publish schemata that will automatically give you
type information for your resource types' fields leaving you free to make
choices about local *policy* assertions - like our example of "all repos need
to state their visibility level explicitly".
