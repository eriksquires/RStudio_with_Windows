# Survival Guide for R on Windows
Erik Squires

# Introduction

I’ve usually used RStudio on OSX or Linux but recent job changes have
forced me to live inside a Windows 10/11 laptop. I wanted to share with
you how to make your R / data science world coexist with Windows.

# Don’t Use Windows Alone

Honestly, the biggest tip is not to attempt to use naked Windows as your
R development platform. You want to use the Windows Subsystem for Linux
(WSL) or a full Linux Virtual Machine. Among the major reasons for this
is consistently good package installation behavior. My experience is
that R packages don’t install reliably in Windows. Some will, some won’t
and why bother fighting it? Just use WSL or a virtual machine with
Ubuntu.

If you haven’t tried it, you may be surprised at how full-featured WSL
is. In addition to acting like Ubuntu, WSL 2 now does a really good job
of supporting GUI based applications thanks to WSLg. I strongly suggest
you use it as your default development location.

One other big advantage of WSL is that because it’s a user space process
you don’t need admin rights to your laptop. So long as you get WSL
version 2 you are pretty much set. The lack of admin rights makes
security people very happy. There are some minor annoyances to walk
through before everything works and we’ll go over them. Except for the
desktop experience (long live Xfce!) you will find WSL a very cozy place
to live. Of course, WSL is also a very happy place for Python
development, so extending WSL to support RStudio seems like a natural
fit. WSL is also a much lower maintenance environment than running
Docker for Windows just to do R development.

## RStudio Server?

Another possibility, outside the scope of this document, is to use
RStudio Server. This is especially useful if your user base tends to
have seriously underpowered laptops. In addition to giving you full
Linux/Rstudio goodness you also have a single place to manage package
dependencies. Your team gains velocity because you aren’t all trying to
figure out which Linux packages must be present to compile a specific R
package. This can also be a huge help in managing connectivity to
corporate data sources. Fixing access issues in one place may be much
easier than fixing it in every developer’s laptop. Imagine having to set
up R with ODBC on half a dozen laptops and you’ll see where this wins.

For the rest of this article we assume you will go down the WSL path,
but some tips may still apply to you elsewhere, like using the Posit
repo and `here()`.

# WSL Configuration

Make sure you have the correct WSL version. You’ll want to have version
2, which *is* also available for Windows 10. In PowerShell:

``` text
PS C:\WINDOWS\system32> wsl -l -v
  NAME      STATE           VERSION
* Ubuntu    Running         2
PS C:\WINDOWS\system32>
```

If you are not on version 2 you’ll need admin rights to run an elevated
PowerShell and then:

``` text
PS C:\WINDOWS\system32> wsl --update
```

To complete your R/WSL installation you’ll need to restart WSL a few
times. For that you’ll use:

``` bash
PS C:\WINDOWS\system32> wsl --shutdown
```

If you are already at version 2 you should be able to install basic
editors and any package really that isn’t network dependent. For
instance:

``` bash
$ sudo apt install gedit
```

Go ahead and install your favorite editor now and replace `gedit` with
it whenever you see it. Another benefit of WSL 2 is you can install
Ubuntu 24, which would not install on WSL 1.

# Git

In addition to git there is now a gh (github) CLI which has a number of
useful features to make your interactions with github easier, but I
especially like the auth command.

``` bash
$ curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
$ sudo apt update
$ sudo apt install git gh

# Make sure you already have a Github login.
# Add --hostname your_gh_server.your_corp.com if using a private repo
$ gh auth login 

$ git config --global user.name "Your Name"
$ git config --global user.email "your.email@example.com"
```

# VS Code

Really can’t stress enough how much I hate VS Code. Mostly because my
use of it is intermittent and this is one of those bro tools that wants
you to show your machismo by absolutely not making anything intuitive.
Still, I do use it, especially with SQL Tools extensions to help me view
and edit my source databases.

This may be confusingly simple because previous versions of WSL and VS
Code required more complicated setups and shims. That’s no longer true.

**Do NOT install VS Code in WSL.**

Install VS Code from the Microsoft Store like any other Windows
application. The trick to a happy life is that we will *install* VS Code
in Windows but we will *launch* VS Code from inside WSL. You can use the
Microsoft Store UI to locate VS Code or you can use PowerShell to
install it like this:

``` text
PS C:\WINDOWS\system32> winget install Microsoft.VisualStudioCode
```

There is one minor fixup in WSL you should do now, before starting code.
Set up the git commit message editor to use `code` as well:

``` bash
$ git config --global core.editor "code --wait"
$ cd ~/dev
```

OK, now you are ready to experience the magic of VS Code / WSL
integration!

``` bash
$ cd ~/dev
$ code . # Launches the Windows VS Code but sees ~/dev file folders.
```

When it finishes launching look on the bottom left corner of the window
and you should see this:

![WSL Connected](images/wsl_connected.jpg)

If you do then you are all set. VS Code will work as if it was natively
running in your WSL / Ubuntu OS. Your VS add-ons, virtual environment
settings, and direnv will all behave exactly as you would expect so long
as you start code in the right directory.

# AppArmor

WSL is meant to look and feel like a terminal only version of real
Ubuntu and for the most part it offers you just that. The one place
where the illusion breaks is with apparmor. Normally you don’t care
until an installer breaks because apparmor really isn’t there. This
applies to most packages which work over the network. One example of a
tool that breaks in installation is the Postgres admin tool,
pgadmin-desktop.

It’s worth just getting this out of the way once so you never have to
deal with it again. It is not required for RStudio.

## Configure systemd

``` bash
$ sudo gedit /etc/wsl.conf
```

Add the following two lines:

``` text
[boot] 
systemd=true
```

Note that this may make the following warning appear each time you start
WSL. It is safe to ignore:

``` text
wsl: Failed to start the systemd user session for 'yourname'. See journalctl for more details.
```

## Edit fstab

Add the following line to the end of /etc/fstab:

``` text
securityfs /sys/kernel/security securityfs defaults 0 0
```

## Restart

``` text
# In powershell:
wsl --shutdown
```

Start your WSL normally.

## Enable AppArmor

``` bash
sudo systemctl enable --now apparmor
```

Reboot WSL once more and try an experiment:

``` bash
$ sudo apt install pgadmin-desktop
```

If this installs without throwing an error you are good to go.

# R

## Install R itself:

``` bash
$ sudo apt install build-essential gfortran
$ sudo apt install r-base
$ sudo apt install libfontconfig1-dev libfreetype6-dev
```

## Configure your R Package Source

Before installing user libraries set up your .Rprofile to use the Posit
repo. The main reason is this repo has pre-compiled R packages so is
much faster than CRAN. Duckdb for instance can take an hour or more to
install from CRAN.

Find your Ubuntu codename:

``` text
$ lsb_release -a

No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.3 LTS
Release:        24.04
Codename:       noble
```

From WSL: `$ gedit ~/.Rprofile` Add these lines to Rprofile. Replace
“noble” with the codename from above then edit the file:

``` r
options(repos = c(CRAN = "https://packagemanager.posit.co/cran/__linux__/noble/latest"))

.libPaths(c("~/R/library", .libPaths()))

# Set Cairo as default bitmap device
# This is only needed for OSX, but here just in case plots don't render.
# options(bitmapType = "cairo")
```

## Install RStudio:

Go to [Posit’s download
page](https://posit.co/download/rstudio-desktop/) and scroll down to
find the right deb package (from lsb_release, above), and paste the link
to it below:

``` bash
$ mkdir ~/downloads
$ cd ~/downloads

$ wget https://download1.rstudio.org/electron/jammy/amd64/rstudio-2025.09.2-418-amd64.deb

$ sudo apt install ./rstudio-2025.09.2-418-amd64.deb
```

Start RStudio and check that you have the Posit repo installed:

``` r
getOption("repos")
```

Should return a string with “posit”.

**Protip:** if you have significant trouble installing R packages from
Posit or CRAN try installing the Ubuntu package first, and then
re-installing your user library version. This will help because the repo
builders do an excellent job of installing all the OS package
dependencies. You can then install the package to your local library.
For instance:

``` bash
$ sudo apt install r-cran-tidyverse
```

Then in RStudio:

``` r
install.packages("tidyverse")
```

This way you’ll get the latest version from CRAN or Posit.

## R Libraries

If you work among others who are forced to do R in Windows I strongly
suggest you leverage the here package/function. It is like `this.path`
but with automatic path separators based on the OS you are using.

here() returns the path relative to the project, so it allows for
portability not just among Windows/Linux but also from R scripts and
Rmd/Qmd docs and Shiny apps.

``` r
library(here)
my_data_file <- here('data', 'my_datastore.rds'))
my_data_file

[1] "/home/yourname/dev/RStudio_with_Windows/data/my_datastore.rds"
```

When running from Rscript though you need to start in the project root.
Here’s an example:

``` bash
$ cd ~/dev/finance_forecast_project
$ Rscript R/my_forecast.R
```

# Homebrew

Homebrew is a great tool for OSX developers. I’m pretty sure every
single development tool and package out there has a recipe or cask for
OSX. It’s being developed for Linux as well but support is pretty spotty
and honestly just IMHO not necessary. ‘apt install’ works just fine.

# Perl

If you are going to do Perl development you should install
[perlbrew](https://perlbrew.pl/). This gives you a personal Perl
installation you can pick and chose the effective Perl version. Install
the padwalker package from CPAN and you should be able to debug in VS
Code like you would any other remote Perl installation.
