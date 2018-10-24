# botor: Reticulate wrapper on 'boto3'

This R package provides access to the 'Amazon Web Services' ('AWS') 'SDK' via the 'boto3' Python module and some convenient helper functions and workarounds, eg taking care of spawning new resources in forked R processes.

## Installation

`botor` is not on CRAN yet, please install from GitHub:

```r
devtools::install_github('daroczig/botor')
```

## Loading the package

Loading the `botor` package might take a while as it will also `import` the `boto3` Python module in the background:

```r
system.time(library(botor))
#>    user  system elapsed 
#>   1.131   0.250   1.191
```

## Getting started

Quick examples:

1. Read a `csv` file stored in S3 using a helper function:

    ```r
    s3_read('s3://botor/example-data/mtcars.csv', read.csv)
    ```

2. Encrypt a string via KMS using a helper function:

    ```r
    kms_encrypt('alias/key', 'secret')
    ```

3. Get AWS user info using the IAM client directly:

    ```r
    iam()$get_user()
    ```

## AWS Auth

The `botor` package by default will use the credentials and related options set in [environmental variables](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html#configuration) or in the `~/.aws/config` and `~/.aws/credentials` files. If you need to specify a custom profile or AWS region etc, there are various options with different complexity and flexibility:

* set the related environment variable(s) before loading `botor`
* call the `botor()` function with the relevant argument to set the config of the default session for the `botor` helper functions, eg

```r
botor(region_name = 'eu-west-42')
```

* start custom sessions via `boto3.session.Session` using the raw `boto3` object from the package, eg

```r
my_custom_session <- boto3$Session(region_name = 'us-west-2')
```

## Using the raw `boto3` module

The `botor` package provides the `boto3` object with full access to the `boto3` Python SDK. Quick example on listing all S3 buckets:

```r
library(botor)
s3 <- boto3$resource('s3')
library(reticulate)
iter_next(s3$buckets$pages())
```

Note that this approach requires a stable understanding of the `boto3` Python module, plus a decent familiarity with `reticulate` as well (see eg `iter_next`) -- so you might want to rather consider using the helper functions described below.

## Using the default `botor` session

Calling `botor()` will provide you a default `boto3` session that is cached internally. You can always override the default session by calling `botor()` again with new arguments. See eg setting the default `boto3` session to use `us-west-2`:

```r
botor(region_name = 'us-west-2')
botor()$resource('s3')
```

A great advantage of using `botor()` instead of custom sessions is that it's fork-safe. See eg:

```r
attr(botor(), 'pid')
#> [1] 31225
attr(botor(), 'pid')
#> [1] 31225

lapply(1:2, function(i) attr(botor(), 'pid'))
#> [[1]]
#> [1] 31225
#>
#> [[2]]
#> [1] 31225

mclapply(1:2, function(i) attr(botor(), 'pid'), mc.cores = 2)
#> [[1]]
#> [1] 13209
#> 
#> [[2]]
#> [1] 13210
```

## Convenient helper functions

The package also provides a `botor` function that returns a pre-initialized default Boto3 session and further R helper functions for the most common AWS actions, like interacting with S3 or KMS. Note, that the list of these functions is pretty limited for now, but you can always fall back to the raw Boto3 functions if needed. PRs on new helper functions are appreciated :)

Examples:

1. Listing all S3 buckets takes some time as it will first initialize the S3 Boto3 client in the background:

    ```r
    system.time(s3_list_buckets())[['elapsed']]
    #> [1] 1.426
    ```

2. But the second query is much faster as reusing the same `s3` Boto3 resource:

    ```r
    system.time(s3_list_buckets())[['elapsed']]
    #> [1] 0.323
    ```

3. Unfortunately, sharing the same Boto3 resource between (forked) processes is not ideal, so `botor` will take care of that by spawning new resources in the forked threads:

    ```r
    library(parallel)
    simplify2array(mclapply(1:4, function(i) system.time(s3_list_buckets())[['elapsed']], mc.cores = 2))
    #> [1] 1.359 1.356 0.406 0.397
    ```

4. Want to speed it up more?

    ```r
    library(memoise)
    s3_list_buckets <- memoise(s3_list_buckets)
    simplify2array(mclapply(1:4, function(i) system.time(s3_list_buckets())[['elapsed']], mc.cores = 2))
    #> [1] 1.330 1.332 0.000 0.000
    ```

The currently supported resources and features via helper functions:

* S3: `s3_list_buckets`, `s3_download_file`, `s3_read`, `s3_upload_file`, `s3_write`
* KMS: `kms_encrypt`, `kms_decrypt`


## Error handling

The convenient helper functions try to suppress the boring Python traceback and provide you only the most relevant information on the error. If you want to see the full tracelog and more details after an error, call `reticulate::py_last_error()`. When working with the raw `boto3` wrapper, you may find `botor:::trypy` useful as well.

```r
s3_download_file('s3://bottttor/example-data/mtcars.csv', tempfile())
#> Error in s3_download_file("s3://bottttor/example-data/mtcars.csv", tempfile()) : 
#>   Python `ClientError`: An error occurred (404) when calling the HeadObject operation: Not Found

s3_read('s3://botor/example-data/mtcars2.csv', read.csv)
#> Error in s3_download(object, t) : 
#>   Python `ClientError`: An error occurred (403) when calling the HeadObject operation: Forbidden

botor(region_name = 'us-west-2')
s3_read('s3://botor/example-data/mtcars.csv', read.csv)
#>     mpg cyl  disp  hp drat    wt  qsec vs am gear carb
#> 1  21.0   6 160.0 110 3.90 2.620 16.46  0  1    4    4
#> 2  21.0   6 160.0 110 3.90 2.875 17.02  0  1    4    4
#> ...
```

## Why the name?

`botor` means "goofy" in Hungarian. This is how I feel when looking back to all the dev hours spent on integrating the AWS Java SDK in R -- this includes `AWR.KMS`, where I ended up debugging and fixing many issues in forked processes, but `AWR.Kinesis` still rocks :)

The name also reminds you that it's not exactly `boto3`, as eg you have to use `$` instead of `.` to access methods.
