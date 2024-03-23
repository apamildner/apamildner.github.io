+++
title = 'The easy way to understand terraform provider aliasing'
date = 2024-03-22T10:10:47+01:00
draft = false
summary = 'This blogg post explains how to use provider aliases and explains some common gotchas'
+++

This is the blogpost I needed to read when I had some problems with configuring my terraform providers and when I really needed to understand how it works. I didn't think the terraform [docs](https://developer.hashicorp.com/terraform/language/providers/configuration) on the topic were concise or provided enough examples so I had a hard time figuring things out. As expected it was quite easy once you know how it works, and here is the resulting blogpost for your convenience.

# Let's start the show with some gotchas
When using providers that's not part of `hashicorp/*` providers, a block like this is needed
```hcl
terraform {
  required_providers {
    restful = {
      source  = "magodo/restful"
      version = "~> 0.9.0"
    }
  }
  required_version = ">= 1.3"
}
```
This is not always obvious, as the default in terraform is to assume you mean `hashicorp/{providername}` of the latest available version when you use a provider. 
This is an example using the `aws` provider, which is then assumed to mean `hashicorp/aws`
```hcl
# Assumes you mean `hashicorp/aws`from hashicorp provider registry
provideer "aws" {

}
```

But you cannot do the following (using this really nice provider for rest API's [restful](https://registry.terraform.io/providers/magodo/restful/latest))
```hcl
provider "restful" {
  base_url = "https://api.X.no"
  security = {
    apikey = [
      {
        in    = "header"
        name  = "apikey"
        value = var.X_API_KEY
      },
    ]
  }
}
```
Because then you will get an error saying
```bash
 Error: Failed to query available provider packages
│ 
│ Could not retrieve the list of available versions for provider
│ hashicorp/restful: provider registry registry.terraform.io does not
│ have a provider named registry.terraform.io/hashicorp/restful
│ 
│ All modules should specify their required_providers so that external
│ consumers will get the correct providers when using a module. To see
│ which modules are currently depending on hashicorp/restful, run the
│ following command:
│     terraform providers

```
I.e the terraform went looking for "hashicorp/restful" and didn't find anything. To fix this, you have to do this:

```hcl
terraform {
  required_providers {
    restful = {
      source  = "magodo/restful"
      version = "~> 0.9.0"
    }
  }
  required_version = ">= 1.5"
}
provider "restful" {
  alias    = "api-X"
  base_url = "https://api.X.no"
  security = {
    apikey = [
      {
        in    = "header"
        name  = "apikey"
        value = var.X_API_KEY
      },
    ]
  }
}
```

# How provider aliasing works

Lets say you have a root module, where you are instantiating a `child_module`

In main.tf, in your root module:
```hcl
terraform {
  required_providers {
    restful = {
      source  = "magodo/restful"
      version = "~> 0.9.0"
    }
  }
  required_version = ">= 1.5"
}
provider "restful" {
  alias    = "api_X"
  base_url = "https://api.x.no"
  security = {
    apikey = [
      {
        in    = "header"
        name  = "apikey"
        value = var.API_X_API_KEY
    ]
  }
}

module "child_module" {
    source "./modules/child"
    providers = {
        restful = restful.api_X
    }
}
```

In your child module main.tf:
```hcl
terraform {
  required_providers {
    restful = {
      source  = "magodo/restful"
      version = "~> 0.9.0"
    }
  }
  required_version = ">= 1.3"
}
# Queries something using the restful provider by default,
# which was overridden by the parent to mean restful.api-X
data "restful_resource" "response_X" {
  id = "/api/inventory/machines"
}
```

The above works fine, since you in the parent module essentially say that when the child module requires `restful`, it actually gets `restful.api_X` from the parent module.


## When to use `configuration_aliases` in the child module
The problem arises when you want to use two different providers in the child module. This is the case if you want to use the `magodo/restful` provider, but you want to query two different API's with different API keys for example. Then you need to do this:


In main.tf, in your root module:
```hcl
terraform {
  required_providers {
    restful = {
      source  = "magodo/restful"
      version = "~> 0.9.0"
    }
  }
  required_version = ">= 1.5"
}
provider "restful" {
  alias    = "api_X"
  base_url = "https://api.X.no"
  security = {
    apikey = [
      {
        in    = "header"
        name  = "apikey"
        value = var.X_API_KEY
      },
    ]
  }
}
provider "restful" {
  alias    = "api_Y"
  base_url = "https://api.Y.no"
  security = {
    apikey = [
      {
        in    = "header"
        name  = "apikey"
        value = var.Y_API_KEY
      },
    ]
  }
}

module "child_module" {
    source "./modules/child"
    provider = {
        restful.api_X = restful.api_X
        restful.api_Y = restful.api_Y
    }
}
```

In your child module main.tf:
```hcl
terraform {
  required_providers {
    restful = {
      source  = "magodo/restful"
      version = "~> 0.9.0"
      configuration_aliases = [restful.api_X, restful.api_Y]
    }
  }
  required_version = ">= 1.5"
}
# The default can now mean two things due to the provider aliasing, and therefore
# we need to be specific here
data "restful_resource" "response_X" {
    provider = {
        restful = restful.api_X
    }
  id = "/api/passwords/foo"
}
data "restful_resource" "response_Y" {
    provider = {
        restful = restful.api_Y
    }
  id = "/api/something/else"
}
```
Now the `response_X` and `response_Y` will be using different providers defined in the root module.


# Summary
In this blogpost we saw some examples of using the more advanced features of terraform provider configuration. We talked about simple use case of providers, and the more advanced provider aliasing features availabile. I hope you learned something!