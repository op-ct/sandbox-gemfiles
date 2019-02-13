# Dualbooting bundler with multiple `Gemfile.lock` files


<!-- vim-markdown-toc GFM -->

* [The issue](#the-issue)
* [bundler 1.x solutions](#bundler-1x-solutions)
  * [:trophy: Using `BUNDLE_GEMFILE=` +  `eval_gemfile()`](#trophy-using-bundle_gemfile---eval_gemfile)
  * [Using `BUNDLE_GEMFILE=` env var setting or `--gemfile` flag](#using-bundle_gemfile-env-var-setting-or---gemfile-flag)
  * [`eval_gemfile()`](#eval_gemfile)
    * [Older alternatives to `eval_gemfile`](#older-alternatives-to-eval_gemfile)
  * [~~`bootboot` bundler plugin~~](#bootboot-bundler-plugin)
  * [`appraisal` gem](#appraisal-gem)
  * [`bundle config --local gemfile Gemfile.local`](#bundle-config---local-gemfile-gemfilelocal)
* [bundler 2 solutions](#bundler-2-solutions)
* [Other fun things](#other-fun-things)
  * [wwtd (local Travis simulator)](#wwtd-local-travis-simulator)

<!-- vim-markdown-toc -->

## The issue

We want to bundle gems in a way that supports multiple platforms or versions of
ruby.  This might translate into different `Gemfile` version rules, and will
almost certainly result in different versions recorded in `Gemfile.lock`.

This is sometimes referred to as "Dualbooting" (as in this [conference talk by
@rafaelfranca][9]).

It seems like what we want is a `Gemfile` and `Gemfile.lock` for each supported
platform/ruby. However, in practice, the actual Gemfile content will be mostly
identical.

This would be useful when used with matrixed tests, like Travis
CI's [build matrix](https://docs.travis-ci.com/user/build-matrix/) keyword
`gemfile`:

```yaml
matrix:
  include:
  - rvm: 2.5
    gemfile: gemfiles/Gemfile.rails-3.2.x
    env: ISOLATED=false
  - rvm: 2.5
    gemfile: gemfiles/Gemfile.rails-3.2.x
    env: ISOLATED=true
```

## bundler 1.x solutions

Approaches to this issue have been discussed and evolved over the years:

- 2018:
  - [Multiple Gemfiles, Multiple Ruby Versions, One Rails][1] (2018/11/12)
  - [Proposal to allow defining different lockfile name not based on Gemfile name #6777][5]
- 2017:
  - [Bundler cop idea: recommend eval_gemfile #3903][6] `rubocop-hq/rubocop` issue
  - [RailsConf 2017: Upgrading a big application to Rails 5 by Rafael França][9]
- 2013:
  - [A second Gemfile and lock? I.e. Gemfile.local, Gemfile.lock.local?][3] (ruby-bundler Google group)
  - [How to use different Gemfiles with Bundler][4]
- 2010:
  - [Multiple lockfiles?][10] (ruby-bundler Google group)


### :trophy: Using `BUNDLE_GEMFILE=` +  `eval_gemfile()`

See `bundle_gemfile/` directory for an example

[Multiple Gemfiles, Multiple Ruby Versions, One Rails][1] reviews the current
options (including the [appraisal][appraisal] gem) and makes a strong case for
just using `BUNDLE_GEMFILE` + `eval_gemfile`:

```sh
BUNDLE_GEMFILE=gemfiles/Gemfile.2.4.5 bundle install
BUNDLE_GEMFILE=gemfiles/Gemfile.2.4.5 rails server -p 4321
```

```
# Contents of Gemfile.2.4.5
ruby "2.4.5"

eval_gemfile "Gemfile"
```

### Using `BUNDLE_GEMFILE=` env var setting or `--gemfile` flag

A classic (2010-2013) solution, mentioned in the `ruby=bunder` Google group
discussion, [_A second Gemfile and lock? I.e. Gemfile.local,
Gemfile.lock.local?_][3].

From the article [How to use different Gemfiles with Bundler][4]:

>  Bundler supports passing an environment variable called `BUNDLE_GEMFILE` to
all of its commands. Here is an example how we can tell it to use a file called
`Gemfile-rails4`:
>
>  `BUNDLE_GEMFILE=Gemfile-rails4 bundle install --path vendor/bundle-rails4`
>
>  You can then run tests in the similar way:
>
>  `BUNDLE_GEMFILE=Gemfile-rails4 bundle exec rake spec`
>




### `eval_gemfile()`

See `eval_gemfile/` directory for an example

The [`eval_gemfile`][2] method has been available since bundler 1.2.0.  It is a
popular choice, but [is not considered part of bundler's public API][6] by its
developers.


From [@segiddins comment][7] in [Bundler cop idea: recommend eval_gemfile #3903][6]:

> Please note that `eval_gemfile` is still (purposefully) undocumented, and
> while we strive to support it, **we do not fully consider it public API** at
> this point

#### Older alternatives to `eval_gemfile`

Before `eval_gemfile`, it was common to load another file with the `Gemfile` like so

From `rubocop-hq/rubocop` issue [Bundler cop idea: recommend eval_gemfile #3903][6] (2017):

```ruby
eval(File.read('Gemfile.local')) if File.exist?('Gemfile.local')
```

Mentioned in [A second Gemfile and lock? I.e. Gemfile.local, Gemfile.lock.local?][3] (ruby-bundler Google group):

```ruby
load "#{File.dirname __FILE__}/Gemfile"
```

From [`phamkykhoi/redmine.tanphat.com`][8]
```ruby
  instance_eval File.read(file), file
```


### ~~`bootboot` bundler plugin~~

[`shopify/bootboot`][bootboot] is a bundler plugin than natively supports dualbooting.

* The sync feature _requires_ bundler >= 1.17
* :x: Only supports two versions


### `appraisal` gem

[`thoughtbot/appraisal][appraisal] is a bundler plugin than natively supports dualbooting.

Adds a new file, `Appraisals`:

```ruby
appraise "rails-3" do
  gem "rails", "3.2.14"
end

appraise "rails-4" do
  gem "rails", "4.0.0"
end
```

These combine (and in the event of a collision, override) the dependencies in
your `Gemfile`.


```yaml
# In .travis.yml
gemfile:
  - gemfiles/3.0.gemfile
  - gemfiles/3.1.gemfile
  - gemfiles/3.2.gemfile
```


### `bundle config --local gemfile Gemfile.local`

## bundler 2 solutions

- bundler 2 has many features that can help
- Bundler 2.0 dropped support for Ruby < 2.3 (no legacy Puppet 4) and RubyGems < 3.0


## Other fun things

### wwtd (local Travis simulator)

Travis simulator: https://github.com/grosser/wwtd

* nicer than [travish](https://github.com/orta/travish)

```
wwtd --local        # Run all gemfiles on current ruby -> get rid of Appraisal
```

[1]: http://engineering.appfolio.com/appfolio-engineering/2018/12/10/multiple-gemfiles-multiple-ruby-versions-one-rails-version
[2]:  http://www.rubydoc.info/github/bundler/bundler/Bundler%2FDsl%3Aeval_gemfile
[3]:  https://groups.google.com/forum/#!topic/ruby-bundler/5zVMtUSNSd0
[4]:  http://semaphoreci.com/blog/2013/11/14/how-to-use-different-gemfiles-with-bundler.html
[5]:  https://github.com/bundler/bundler/issues/6777
[6]:  https://github.com/rubocop-hq/rubocop/issues/3903
[7]:  https://github.com/rubocop-hq/rubocop/issues/3903#issuecomment-272712779
[8]:  https://github.com/phamkykhoi/redmine.tanphat.com/blob/e7f473c34a547f667c584b101654a5e02f0cd1fc/Gemfile#L13
[9]:  https://www.youtube.com/watch?v=I-2Xy3RS1ns&t=368s
[10]:  https://groups.google.com/forum/#!msg/ruby-bundler/F_i3HC5faCM/p9zso8v06ZkJ
[appraisal]: https://github.com/thoughtbot/appraisal
[bootboot]: https://github.com/shopify/bootboot
