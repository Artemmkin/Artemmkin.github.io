---
layout: post
title: >
    How to test Terraform built-in functions locally. 
tags: [terr]
---

Terraform has a bunch of [built-in functions](https://www.terraform.io/docs/configuration/interpolation.html#built-in-functions) that allow to perform common operations when writing infrastructure code. Some of them are so common in many programming languages, that you can guess what they are for even without reading the documentation. For example, you'll probably recognize the [length()](https://www.terraform.io/docs/configuration/interpolation.html#length-list-) function which returns the number of elements in a given list or map, the [list()](https://www.terraform.io/docs/configuration/interpolation.html#list-items-) which returns a list consisting of arguments given to a function and the [join()](https://www.terraform.io/docs/configuration/interpolation.html#basename-path-) which joins a list with the delimiter for a resulting string. You can look through the whole list of these functions in the [documenation](https://www.terraform.io/docs/configuration/interpolation.html#built-in-functions).

There is a question though. How do you try out these built-in functions locally and see how they work? In this post, I'm going to show a couple of ways you can do that...
<!--break-->

### Terraform console

Many programming language such as Python and Ruby provide an **interactive console** as a quick way to try out commands and test pieces of code without creating a file. 

Terraform has an interactive console, too. But this console is sort of limited in functionality. It only allows to test [interpolations](https://www.terraform.io/docs/configuration/interpolation.html).

Using `interpolations` you can insert into strings different values which stay unknown before the terraform starts to run. For example, you will often interpolate the value of an input variable into the resource configuration:

```
resource "aws_s3_bucket" "main" {
  bucket = "${var.bucket_name}"
}
``` 

The syntax for interpolation is similar what you can meet in other programming languages. The interpolated value is put inside the curly braces and prefixed with a dollar sign like this `${var.bucket_name}`.

`terraform console` simply creates the interpolation environment. Everything you type into the console is effectively the same as putting it inside `${}` in your configuration files. For example, if you launch a terraform console and type a math expression like `1 + 2`:

```bash
$ terraform console
> 1 + 2
3
```

You will get `3` in the output. This means that you can wrap the `1 + 2` expression inside `${}`, put it in your configuration file and be confident that this will be evaluated to `3`. For example, the following will create `3` S3 buckets in AWS Cloud:

```
resource "aws_s3_bucket" "main" {
  count  = "{1 + 2}"
  bucket = "test-bucket-${count.index}"
}
```

Now that we understand what functionality `terraform console` provides, we can use it to test interpolating the built-in functions.

```
$ terraform console
> list("hello", "world")
[
  hello,
  world
]
> upper("hello world")
HELLO WORLD
```

But remember, because everything you type will be immediately evaluted, you can't create variables in interactive console. Thus, if you need to pass a map or a list to your function, you'll need to create this map or list using [map()](https://www.terraform.io/docs/configuration/interpolation.html#map-key-value-) and [list()](https://www.terraform.io/docs/configuration/interpolation.html#list-items-) functions respectively:

```
> length(list("hello", "world"))
2
> lookup(map("id", "1","message", "hello world" ), "id")
1
> lookup(map("id", "1","message", "hello world" ), "message")
hello world
> lookup(map("id", "1","message", "hello world" ), "author", "None")
None
```

If you intentionally mistype a function name, you can see the inner workings of terraform console. It wraps everything you type inside `${}` and then evaluates the expression:

```
> lengt(list("hello", "world"))
1:3: unknown function called: lengt in:

${lengt(list("hello", "world"))}  <-- the function gets put inside ${}
```

### Output variables

If you miss creating variables and find working with the interactive console confusing, you can try out the built-in functions using output variables.

Create a configuration file for testing. And put some provider in it such as `local` which doesn't have required parameters. [Terraform won't run without a provider component, that is why I use the one that has the least required configuration.]

Then define any variables you want with a `variable` block and put the expression you want to test inside definition of an [output variable](https://www.terraform.io/intro/getting-started/outputs.html).

To demonstrate the same examples with the [length()](https://www.terraform.io/docs/configuration/interpolation.html#length-list-) and [lookup()](https://www.terraform.io/docs/configuration/interpolation.html#lookup-map-key-default-) function using a new approach, I will create the following `test.tf` file:

```
provider local {}

variable "my_list" {
  default = ["hello", "world"]
}

variable "my_map" {
  default = {
    id      = "1"
    message = "hello world"
  }
}

## functions to test
output "my_list_test1" {
  value = "${length(var.my_list)}"
}

output "my_map_test1" {
  value = "${lookup(var.my_map, "id")}"
}

output "my_map_test2" {
  value = "${lookup(var.my_map, "message")}"
}

output "my_map_test3" {
  value = "${lookup(var.my_map, "author", "None")}"
}
```

If I run `terraform apply`, I will get the same results as before:

```bash
$ terraform init
$ terraform apply
Outputs:

my_list_test1 = 2
my_map_test1 = 1
my_map_test2 = hello world
my_map_test3 = None
```

P.S. You find the `test.tf` file as well as an example for testing terraform templates in my [GitHub repo](https://github.com/Artemmkin/terraform-local-test).
