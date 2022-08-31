# Heroku Buildpack: R

[![CI](https://github.com/virtualstaticvoid/heroku-buildpack-r-build2/actions/workflows/main.yml/badge.svg)](https://github.com/virtualstaticvoid/heroku-buildpack-r-build2/actions/workflows/main.yml)
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fvirtualstaticvoid%2Fheroku-buildpack-r.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Fvirtualstaticvoid%2Fheroku-buildpack-r?ref=badge_shield)

This is a [Heroku Buildpack][buildpacks] for applications which use [R][rproject] for
statistical computing and [CRAN][cran] for R packages.

The buildpack supports the [heroku-18][stack18][^18support], [heroku-20][stack20][^20support]
and [heroku-22][stack22][^22support] stacks.

It also includes support for the [Packrat][packrat] and [renv][renv] package managers, and
the [Shiny][shiny] and [Plumber][plumber] web application frameworks.

## Usage

The buildpack's name is [`vsv/heroku-buildpack-r`][bpurl]. Provide it when creating your
application on Heroku as follows:

```bash
heroku create --buildpack vsv/heroku-buildpack-r
```

You can add it to an existing application using the `buildpacks:add` command, as follows:

```bash
heroku buildpacks:add vsv/heroku-buildpack-r
```

Alternatively, you can use the Git URL of this repository, together with the branch name.

```
https://github.com/virtualstaticvoid/heroku-buildpack-r.git#main
```

The buildpack will detect your application makes use of R if it has one (or more)
of the following files in the project directory:

* `init.R`
* `packrat/init/R`
* `renv/activate.R`
* `run.R`
* `app.R`
* `plumber.R`

If the `init.R` file is provided, it will be executed in order to install any R packages,
and if `packrat/init.R` or `renv/activate.R` files are found, the respective package manager
will be bootstrapped and packages installed.

Additionally:

* If the `run.R` file is provided, the buildpack will be configured as a Shiny application.
* If the `plumber.R` file is provided, the buildpack will be configured as a Plumber application.

See the [detect](bin/detect) script for the matching logic used.

### Installing R Packages

The `init.R` file is used to install R packages as required.

*NOTE:* Using either [Packrat][packrat] or [renv][renv] are a better way to manage your
package dependencies and their respective versions, so the `init.R` file isn't required
if you use `packrat` or `renv`.

The following example `init.R` file can be used. Provide the package names you want to
install to the `my_packages` list variable:

```R
# init.R
#
# Example R code to install packages if not already installed
#

my_packages = c("package_name_1", "package_name_2", ...)

install_if_missing = function(p) {
  if (p %in% rownames(installed.packages()) == FALSE) {
    install.packages(p, clean=TRUE, quiet=TRUE)
  }
}

invisible(sapply(my_packages, install_if_missing))
```

R packages can also be installed by providing a `.tar.gz` package archive file, if a
specific version is required, or it is not a publicly published package. See
[local-packages](test/local-packages) for an example.

```R
# init.R
#
# Example R program to installed package from local path
#

install.packages("PackageName-Version.tar.gz", repos=NULL, type="source")
```

*NOTE:* The path to the package archive should be a relative path to the project root
directory, so that it works locally in development and during deployment on Heroku.

#### R Package Installation Helper

For convenience, a R helper function, [`helpers.installPackages`](bin/helpers.R), is
included by the buildpack to make installing packages easier.

Thus the `init.R` file can be reduced to a single line of R code as shown. Provide the
package names you want to install as arguments to the helper:

```R
helpers.installPackages("package_name_1", "package_name_2", ...)
```

### Installing Binary Dependencies

This version of the buildpack still supports the use of an `Aptfile` for installing
additional system packages, however this functionality is going to be _deprecated_ in
future as it isn't a foolproof solution.

It is based on the same technique as used by the [heroku-buildpack-apt][bpapt] buildpack
to install Ubuntu packages using `apt-get`.

There are various technical and security reasons why it is no longer recommended,
so your mileage may vary.

If any of your R packages dependend on system libraries which aren't
[included by Heroku][herokupkgs], such as `libgmp`, `libgomp`, `libgdal`, `libgeos`
and `libgsl`, you should use the Heroku [container stack][container-stack] together
with [heroku-docker-r][heroku-docker-r] instead.

## R Applications

### Heroku Console

You can run the R console application as follows:

```bash
$ heroku run R ...
```

Type `q()` to exit the console when you are finished.

You can also run the `Rscript` utility as follows:

```bash
$ heroku run Rscript ...
```

_Note that the Heroku slug is read-only, so any changes you make during the session will be lost._

### Shiny Applications

Shiny applications must provide a `run.R` file, and can also include an `init.R` in order to
install additional R packages. The Shiny package does not need to be installed, as it is
included in the buildpack already.

The `run.R` file should contain at least the following code, in order to run the web application.

Notice the use of the `PORT` environment variable, provided by Heroku, which is used
to configure Shiny and the host must be `0.0.0.0`.

```R
# run.R
library(shiny)

port <- Sys.getenv('PORT')

shiny::runApp(
  appDir = getwd(),
  host = '0.0.0.0',
  port = as.numeric(port)
)
```

See the [virtualstaticvoid/heroku-shiny-app][shiny-app] example application.

### Plumber Applications

Plumber applications must provide an `app.R` file, but can also include an `init.R` in order
to install additional R packages. The Plumber package does not need to be installed, as it
is included in the buildpack already.

The `app.R` file should contain at least the following code, in order to run the web application.

Notice the use of the `PORT` environment variable, provided by Heroku, which is used to
configure Shiny and the host must be `0.0.0.0`.

```R
# app.R
library(plumber)

port <- Sys.getenv('PORT')

server <- plumb("plumber.R")

server$run(
  host = '0.0.0.0',
  port = as.numeric(port)
)
```

See the [virtualstaticvoid/heroku-plumber-app][plumber-app] example application.

### Recurring Jobs

You can use the [Heroku scheduler][scheduler] to schedule a recurring R process.

An example command for the scheduler to run `prog.R`, would be `R --file=prog.R --gui-none --no-save`.

## Technical Details

### R Versions

The default R version can be overridden by setting the `R_VERSION` environment variable.

```bash
heroku config:set R_VERSION=4.0.0
```

The following table lists the available combinations of Heroku Stack and R version.
They are [built][build2] periodically as and when the [Debian R packages][cloud-r-project]
are available.

| R / Stack | `18` | `20` | `22` |
|:---------:|:----:|:----:|:----:|
| `3.6.3`   |  ⬦   |      |      |
| `4.0.0`   |  ⬦   |   ⬦  |      |
| `4.0.5`   |  ⬦   |   ⬦  |      |
| `4.1.2`   |  ⬦   |   ⬦  |  ⬦   |
| `4.1.3`   |  ⬦   |   ⬦  |  -   |
| `4.2.0`   |  ⬥   |   -  |  ⬦   |
| `4.2.1`   |      |   ⬥  |  ⬥   |

Legend:

* `⬥` = default version for given stack
* `⬦` = available
* `-` = no package available

### Slug Compilation vs Runtime use of `chroot`

This version of the buildpack still uses a [fakechroot][fakechroot] during slug compilation,
to compile R packages which may include C or Fortran code. However it no longer uses the
`chroot` at runtime so it can work better in scenarios where other language buildpacks are
used, such as with Python, Ruby or Java, and so that the [slug size][slugsize] is greatly reduced.

If you are migrating to this version of the buildpack, you no longer need to prefix commands
to use `fakechroot`, `fakeroot` or `chroot`. Wrappers of these commands are included and
they will output warning messages to alert you of their use.

### Buildpack Binaries

The binaries used by the buildpack are hosted on AWS S3 at [https://heroku-buildpack-r.s3.amazonaws.com][s3].

See the [heroku-buildpack-r-build2][build2] repository for building the buildpack binaries yourself.

### Process Types

The buildpack includes the following default process types:

* `console`: Executes the `R` terminal application, which is typically used for debugging.
* `web`: Executes `run.R` to run Shiny or Plumber applications.

The `R` and `Rscript` executables are available like any other executable, via the `heroku run` command.

#### Procfile

You can include a [Procfile][procfile] in your project if you want to override the default process
types and/or their command lines. This is typically required if you are using multiple buildpacks.

For example, the following Profile defines the commands for the `web` and `console` processes.

```
web: R --file=myprogram.R --gui-none --no-save
console: R --no-save
```

#### `heroku.yml`

You can include the [`heroku.yml`][herokuyml] build manifest in you project if you want to
override the default process types and/or their command lines, within the `run` section.
This is typically required if you are using multiple buildpacks.

For example, the following `heroku.yml` file defines the commands for the `web`
and `console` processes.

```yaml
run:
  web: R --file=myprogram.R --gui-none --no-save
  console: R --no-save
```

### Paths

Where possible, always use relative paths for files, so that your application is more portable;
so that it can run locally in development and at runtime on Heroku without any differences.

The current directory on Heroku will always be `/app` and your application will be installed
to this directory, so relative paths should be in respect of the root directory of your project.

If you need to use absolute paths, consider using [`getwd()`][getwd] together with
[`file.path()`][filepath] to build up the path instead of hardcoding them.

### `.Rprofile`

You can include an [`.Rprofile`][rprofile] in your application's root directory and it will
be executed at the start of any R process.

It can be used as a convenient way to bootstrap your application, sourcing common utilities or
performing configuration tasks.

Please _do not_ use it to install R packages, since it may cause problems during deployment
(slug compilation) and  _will_ fail at runtime.

### CRAN Mirror Override

It is possible to override the default CRAN mirror used, by providing the URL via
the `CRAN_MIRROR` environment variable.

E.g. Override the URL by setting the variable as follows.

```bash
heroku config:set CRAN_MIRROR=https://cloud.r-project.org/
```

Check the CRAN [mirror status][mirrors] page to ensure the mirror is available.

### Caching

To improve the time it takes to deploy the buildpack caches the R binaries and installed R packages.

If you need to purge the cache, it is possible by using [heroku-repo][heroku-repo] CLI plugin
via the `heroku repo:purge_cache` command.

See the [purge-cache][purge] documentation for more information.

## Build Output Verbosity

In previous versions of the buildpack, the full output of `install.packages()` was emitted
during slug compilation, which lead to very verbose output and made it hard to spot issues
in some instances. Packages are now installed with `quiet=TRUE` option set by default.

To restore the previous behaviour, set the `PACKAGE_INSTALL_VERBOSE` environment variable
to a value of `1` before deploying your application:

```bash
heroku config:set PACKAGE_INSTALL_VERBOSE=1
```

To revert the setting use the `config:unset` command:

```bash
heroku config:unset PACKAGE_INSTALL_VERBOSE
```

## Hacking

To enable debug outputs during deployment, set the `BUILDPACK_DEBUG` environment variable.

```bash
heroku config:set BUILDPACK_DEBUG=1
```

## Credits

* Original inspiration from [Noah Lorang's Rook on Heroku][rookonheroku] project.
* Script snippets from the [rstudio/r-builds][r-builds] project.
* Tests from the [rstudio/r-docker][r-docker] project.
* [fakechroot][fakechroot] library.
* [tcl/tk][tcltk] library.

## License

MIT License. Copyright (c) 2020 Chris Stefano. See [LICENSE](LICENSE) for details.

<!-- footnotes -->

[^18support]: The Heroku-18 stack is _deprecated_ and will reach end-of-life on April 30th, 2023.
[^20support]: Heroku-20 is based on Ubuntu 20.04. It will be supported through April 2025.
[^22support]: Heroku-22 is based on Ubuntu 22.04. It will be supported through April 2027.

<!-- links -->

[bpapt]: https://elements.heroku.com/buildpacks/heroku/heroku-buildpack-apt
[bpurl]: https://elements.heroku.com/buildpacks/virtualstaticvoid/heroku-buildpack-r
[build2]: https://github.com/virtualstaticvoid/heroku-buildpack-r-build2
[buildpacks]: https://devcenter.heroku.com/articles/buildpacks
[cloud-r-project]: https://cloud.r-project.org/bin/linux/ubuntu/
[container-stack]: https://devcenter.heroku.com/categories/deploying-with-docker
[cran]: https://cran.r-project.org
[fakechroot]: https://github.com/dex4er/fakechroot/wiki
[filepath]: https://www.rdocumentation.org/packages/base/versions/3.6.0/topics/file.path
[getwd]: https://www.rdocumentation.org/packages/base/versions/3.6.0/topics/getwd
[heroku-docker-r]: https://github.com/virtualstaticvoid/heroku-docker-r
[heroku-repo]: https://github.com/heroku/heroku-repo
[herokupkgs]: https://devcenter.heroku.com/articles/stack-packages
[herokuyml]: https://devcenter.heroku.com/articles/build-docker-images-heroku-yml#run-defining-the-processes-to-run
[mirrors]: https://cran.r-project.org/mirmon_report.html
[packrat]: https://rstudio.github.io/packrat
[plumber-app]: https://github.com/virtualstaticvoid/heroku-plumber-app
[plumber]: https://www.rplumber.io
[procfile]: https://devcenter.heroku.com/articles/procfile
[purge]: https://github.com/heroku/heroku-repo#purge-cache
[r-builds]: https://github.com/rstudio/r-builds
[r-docker]: https://github.com/rstudio/r-docker
[renv]: https://rstudio.github.io/renv/
[rookonheroku]: https://github.com/noahhl/rookonheroku
[rprofile]: https://cran.r-project.org/doc/manuals/r-release/R-intro.html#Customizing-the-environment
[rproject]: https://www.r-project.org
[s3]: https://heroku-buildpack-r.s3.amazonaws.com
[scheduler]: https://addons.heroku.com/scheduler
[shiny-app]: https://github.com/virtualstaticvoid/heroku-shiny-app
[shiny]: https://shiny.rstudio.com
[slugsize]: https://devcenter.heroku.com/articles/slug-compiler#slug-size
[stack18]: https://devcenter.heroku.com/articles/heroku-18-stack
[stack20]: https://devcenter.heroku.com/articles/heroku-20-stack
[stack22]: https://devcenter.heroku.com/articles/heroku-22-stack
[tcltk]: https://www.tcl.tk


[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fvirtualstaticvoid%2Fheroku-buildpack-r.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Fvirtualstaticvoid%2Fheroku-buildpack-r?ref=badge_large)