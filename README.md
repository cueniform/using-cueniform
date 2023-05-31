# Using Cueniform

## Welcome to Cueniform!

Cueniform is a project that's
trying to make it easier and safer
to configure
Hashicorp's
[Terraform](https://terraform.io)
infrastructure automation tool.
It does this by using the awesome power of the
[CUE](https://cuelang.org)
data validation language.

Cueniform has some big plans and ideas for the future,
but they're not all in place just yet.

Until they are,
this documentation has 2 main goals:

1. to show what's possible, today, using Cueniform, and
1. to point towards what *will* be possible soon!

## What's possible today with Cueniform

Using Cueniform, you can:

- [create Terraform configurations in CUE](configuring-terraform-with-cue-using-cueniform.md),
  and validate that their core structure matches what Terraform expects
- [validate the core structure of Terraform JSON configurations](https://github.com/cueniform/terraform-core-schemata/blob/main/README.md)
  - [JSON syntax documentation](https://developer.hashicorp.com/terraform/language/v1.4.x/syntax/json)

## What's going to be possible with Cueniform soon

We're working hard on bringing features to Cueniform that will allow you to:

- validate the structure and fields of all Official and Partner providers on the
  [Terraform Registry](https://registry.terraform.io/browse/providers)
- validate Terraform runtime expressions
- apply policies that check your configurations are adhering to industry best
  practices
- apply custom policies to your configuration
- share custom policies with other users
