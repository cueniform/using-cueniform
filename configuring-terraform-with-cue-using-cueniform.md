# Configuring Terraform with CUE using Cueniform

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

1. Teach Cueniform about your Terraform version

   ```shell
   cat <<HERE >cnifm.cue; cue fmt
   package cueniform
   import (
     "cueniform.com/x/terraform-core/1.4.x/core"
   )
   configuration: core.#Configuration
   HERE
   ```

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
   cue export .:cueniform --expression configuration --force --outfile config.tf.json
   ```

1. Run Terraform

   ```shell
   terraform apply -auto-approve
   ```

1. Version control your important files:

   - `cnifm.cue`
   - `terraform.cue`
   - `config.tf.json`
   - the contents of `cue.mod`
     - you might prefer to use a git-submodule rather than commiting the directory
     - CUE and Cueniform don't care how you deal with this!
