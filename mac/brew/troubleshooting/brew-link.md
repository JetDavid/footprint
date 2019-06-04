# An error occurred during the 'Brew link' step

## Problem

在执行brew install时，可能会遇到如下错误。

```bash
==> Installing vim dependency: xz
==> Downloading https://homebrew.bintray.com/bottles/xz-5.2.4.mojave.bottle.tar.gz
==> Downloading from https://akamai.bintray.com/01/010667293df282c8bceede3bcd36953dd57c56cef608d09a5b50694ab7d4b96b?__gda__=exp=1557284546~hmac=a68b245ad528d020de62748712f17de4937c4
######################################################################## 100.0%
==> Pouring xz-5.2.4.mojave.bottle.tar.gz
Error: The `brew link` step did not complete successfully
The formula built, but is not symlinked into /usr/local
Could not symlink include/lzma
/usr/local/include is not writable.

You can try again using:
  brew link xz
==> Summary
🍺  /usr/local/Cellar/xz/5.2.4: 92 files, 1MB
==> Installing vim dependency: python
==> Downloading https://homebrew.bintray.com/bottles/python-3.7.3.mojave.bottle.tar.gz
==> Downloading from https://akamai.bintray.com/25/25e0099852136c4ef1efd221247d0f67aa71f7b624211b98898f8b46c612f40d?__gda__=exp=1557284552~hmac=08c5ccf700336a358c6d1d000962b1590a145
######################################################################## 100.0%
==> Pouring python-3.7.3.mojave.bottle.tar.gz
Error: An unexpected error occurred during the `brew link` step
The formula built, but is not symlinked into /usr/local
Permission denied @ dir_s_mkdir - /usr/local/Frameworks
Error: Permission denied @ dir_s_mkdir - /usr/local/Frameworks
```

## Solution 1

```bash
brew doctor
Please note that these warnings are just used to help the Homebrew maintainers
with debugging if you file an issue. If everything you use Homebrew for is
working fine: please don't worry or file an issue; just ignore this. Thanks!

Warning: The following directories do not exist:
/usr/local/include
/usr/local/sbin

You should create these directories and change their ownership to your account.
  sudo mkdir -p /usr/local/include /usr/local/sbin
  sudo chown -R $(whoami) /usr/local/include /usr/local/sbin

Warning: You have unlinked kegs in your Cellar.
Leaving kegs unlinked can lead to build-trouble and cause brews that depend on
those kegs to fail to run properly once built. Run `brew link` on these:
  gdbm
  python
  xz
  lua
```

## Solution 2

brew doctor

than

sudo chown -R $(whoami) $(brew --prefix)/*

note the $(brew --prefix)/* ...High Sierra doesn't allow you to change permissions on /user/local directly)

then this:

sudo install -d -o $(whoami) -g admin /usr/local/Frameworks

## Reference

[Error: An unexpected error occurred during the `brew link` step](https://github.com/jakubroztocil/httpie/issues/645)