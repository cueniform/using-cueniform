# Configuring Terraform with CUE using Cueniform

## Quick start

Here are some steps to get you started:

1. [Install CUE](https://cuelang.org/docs/install/)
   version `v0.6.0-alpha.1` or later

1. Initialise your local CUE module

   ```shell
   cue mod init
   ```

1. Clone some Cueniform repos

   ```shell
   git clone \
       https://github.com/cueniform/terraform-core-schema \
       cue.mod/pkg/cueniform.com/x/terraform-core
   ```

1. Teach Cueniform about your Terraform version, and how it should validate resources and data-sources

   ```shell
   cat <<HERE >cnifm.cue; cue fmt
   package cueniform
   import (
     "cueniform.com/x/terraform-core/1.4/core"
   )
   configuration: core.#Configuration & {
     resource?: [Type=_]: [_]: #Entities[Type].#Resource
     data?: [Type=_]: [_]:     #Entities[Type].#DataSource
   }
   #Entities: _
   HERE
   ```

1. Teach Cueniform about the built-in provider

   ```shell
   package cueniform
   import (
     "cueniform.com/x/terraform-core/1.4/providers/builtin"
   )
   #Entities: {
     builtin.#Entities
   }
   ```

   (The `#Entities` namespace is where we'll be referencing all the upstream
   provider schemata published by Cueniform, in the the future. But right now
   you'll need to generate *manual* schemata for all the resource and data-source
   types you use. Don't worry: a default, opinion-less schema is trivial to
   generate in CUE, so you only need to dive into provider documentation on the
   Terraform Registry *if you want to*! We'll see
   [how to do this](#teaching-cueniform-about-new-resources-and-data-sources)
   shortly ...)

1. Start configuring Terraform

   ```shell
   cat <<HERE >terraform.cue; cue fmt
   package cueniform
   configuration: {
     output: hello: {
       value: "Hello from Cueniform!"
     }
   }
   HERE
   ```

1. Export your config into Terraform JSON

   ```shell
   cue export .:cueniform -e configuration -f -o config.tf.json
   ```

1. Run Terraform

   ```shell
   terraform apply -auto-approve
   ```

1. Version control your important files:

   - `cnifm.cue`
   - `providers.cue`
   - `terraform.cue`
   - `config.tf.json`
   - the contents of `cue.mod`
     - you might prefer to use a git-submodule rather than commiting the directory
     - CUE and Cueniform don't care how you deal with this!

## Teaching Cueniform about new resources and data-sources

These lines, in `cnifm.cue`, instruct CUE that every resource and data-source
has a matching schema, inside the `#Entities` namespace, and that each resource
or data-source must adhere to its matching schema:

```cue
configuration: core.#Configuration & {
  resource?: [Type=_]: [_]: #Entities[Type].#Resource
  data?: [Type=_]: [_]:     #Entities[Type].#DataSource
}
```

Letting you expanding that `#Entities` namespace automatically is part of the
work that we're doing with Cueniform, behind the scenes. But until that's ready,
you need to choose between:

1. Manually teaching Cueniform about the schemata of the resources and
   data-sources you're using
1. Replacing these 4 lines with `configuration: core.#Configuration`

Either choice is valid, but the quicker, easier option (#2!) leaves you
**without** a way of adding important assertions, later on. Essentially,
choosing option #2 means that CUE will **only** be able to validate the
structure of the top-level configuration keys that Terraform expects (e.g.
`terraform`), and won't be asserting *anything* about the resources you're
creating!

Instead of choosing option #2, here's how to get started quickly with option
#1. It's *really* simple and easy ...

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
cat <<HERE >custom-schemata.cue; cue fmt
package cueniform
#Entities: {
  github_repository: #Resource: {...}
}
HERE
```

That `...` is CUE's way of saying "any fields are permitted here".

Now we can add `github_repository` entries in `terraform.cue`, and CUE will be happy:

```cue
cat <<HERE >>terraform.cue; cue fmt
configuration: resource: {
  github_repository: {
    my_first_repo: {
      name: "a_repo"
    }
  }
}
> HERE

cue export .:cueniform -e configuration -f -o config.tf.json
```

But ... **what's the *point* of adding a schema** that doesn't impose any
opinions?

Well, first of all, it's implicitly checking our resource type spelling! If we
mistype the `github_repository` resource type anywhere (e.g. as `github_repo`),
then CUE will tell us.

But more importantly, let's imagine that at some point in the future, we make a
mistake in our configuration. And after we realise what we've done wrong and
fix it, we want to add a schema assertion and a policy assertion to prevent us
from making this mistake ever again. That file is the right place to put these
assertions:

```shell
cat <<HERE >custom-schemata.cue; cue fmt
package cueniform
#Entities: {
  github_repository: #Resource: {
    name!: string
    visibility!: "public" | "private"
    ...
  }
}
HERE
```

Now all our `github_repository` resources (a) have to have a `name` field and
(b) have to *explicitly* state if the repos they create are "public" or
"private" (without relying on the `github` provider's implicit default).

CUE lets us build up these assertions bit by bit, over time: putting effort in
where it brings us value, and being lazy where *that*'s valuable :-)

Ultimately, Cueniform will publish schemata that will automatically give you
type information for your resource types' fields (the `name!: string`, above),
leaving you free to make choices about local *policy* assertions - like our
example of "all repos need to state their visiblity level explicitly".
