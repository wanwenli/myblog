---
layout: post
title: "Why I used Jekyll-bootstrap instead of Octopress"
category: programming
excerpt: My bad experience with Octopress
---
I was amazed by the beautiful blog made by my friend.
He suggested me use `Octopress` or `Jekyll`.
However I faced a lot of problems when I tried to install both of them.

Below are some examples:

##### Problem 1

    Building native extensions.  This could take a while...
    ERROR:  Error installing jekyll:
    ERROR: Failed to build gem native extension.

    /usr/bin/ruby1.9.1 extconf.rb
    /usr/lib/ruby/1.9.1/rubygems/custom_require.rb:36:in `require': cannot load such file -- mkmf (LoadError)
    from /usr/lib/ruby/1.9.1/rubygems/custom_require.rb:36:in `require'
    from extconf.rb:1:in `<main>'

##### Solution 1

	sudo apt-get install ruby1.9.1-dev

##### Problem 2  

	Bundler::GemfileNotFound

##### Solution 2
find the Gemfile by command:

    locate Gemfile

cd into the directory

##### Problem 3

    rake aborted!
    You have already activated rake 0.9.6, but your Gemfile requires rake 0.9.2.2.

##### Solution 3
run command

    $ bundle exec rake install

Apply to all the incidents when rake is aborted.

Finally the big boss came and I was unable to configure the rake such that it can
point to my github local repo. Sad...

Now I switch to `Jekyll Bootstrap`. Hopefully it is a better solution.
Hopefully it was not because I was noob.
