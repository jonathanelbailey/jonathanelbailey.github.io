---
title: "Up and Running With Jekyll and GitHub"
modified:
categories:
excerpt: 'Learn how to serve up a git page using the rich features of Jekyll 3.0'
tags:
  - git
  - jekyll
  - ruby
  - development
header:
  overlay_image: 20151010_113344.jpg
  caption: "Photo credit: Jonathan Bailey"
date: 2016-05-16T21:53:40-05:00
---
{% include toc %}

{% include base_path %}

## Introduction

This guide explains how to work with Jekyll.

### What is Jekyll?

[Jekyll](http://jekyllrb.com/) is a tool that allows for the creation of static websites.  To get started, it can be as easy as cloning a site template repository from github, or you can run `jekyll new 'my-site'` to start from scratch.  In this example, I'm going to be using the [minimal mistakes](https://mademistakes.com/work/minimal-mistakes-jekyll-theme/) to create my github page, and I'll be running my dev environment on Centos 7.  My workstation is running Windows 10, and since jekyll doesn't play nice with Windows, it's recommended that you use a linux powered VM or OSX.

## Building Your Dev Environment

First, we'll need to enable the EPEL repo from Fedora by running `yum install epel-release` with root.  Next, we'll need to install the ruby development environment:

{% highlight bash %}
yum install ruby ruby-devel nodejs httpd git
{% endhighlight %}

and finall, run `gem install jekyll` to install the libraries.

### Configuring Ruby

since you'll likely want to wrangle your ruby environment, it's recommended to use 'rbenv' to handle your ruby dependencies if you're running multiple versions on your workstation.  run `gem install rbenv` to install the libraries, and create a directory that you're going to use specifically for jekyll site work.  In this example, we'll create the ~/sites folder:

{% highlight bash %}
mkdir ~/sites
cd sites
rbenv install 2.3.1
{% endhighlight %}

## Configuring your Repo

The next step requires that you have a github account.  Ensure that you're logged in to your account, and go to the [minimal mistakes repo](https://github.com/mmistakes/minimal-mistakes).  In the top right hand corner, click 'Fork'.

{% capture fig_img %}
![fork the repo]({{ base_path }}/images/fork.png)
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>Fork the repo.</figcaption>
</figure>

Next, you'll need to go to your new repo, and rename it to `YOURUSERNAME.github.io`, by changing the repo name under the 'settings' tab.  once you've forked the repo, it's time to clone it at your new dev machine.  In this example, I'm using my own username to clone the repo:

{% highlight bash %}
cd ~/sites
git clone https://jonathanelbailey/jonathanelbailey.github.io.git
cd jonathanelbailey.github.io
git checkout gh-pages
bundle install
{% endhighlight %}

`bundle install` should begin to install all dependencies related to the jekyll template we're using.
**Note:** because the Minimal Mistakes theme is a project, Michael Rose uses a branch called 'gh-pages' as the branch that will run his project page.  In this guide, we'll be creating a user page.  For more information, check out [Github's article](https://help.github.com/articles/user-organization-and-project-pages/) explaining more on gitpages.
{: .notice--info}

### Create a Working Branch

Once bundler completes its installation of all dependencies, you'll want to create a working branch.  You'll want to make sure your branch is a branch of the gh-pages branch:

{% highlight bash %}
git branch working-branch
git checkout working-branch
{% endhighlight %}

## Working With Jekyll

Next, you'll want to see how jekyll works.  Let's serve up the current page, which should just be the Minimal Mistakes live demo.

{% highlight bash %}
jekyll serve --host=jekylldev --config=\_config.yml,\_config.dev.yml
{% endhighlight %}

where `jekylldev` is the name of my development machine.  Then we can open up `http://jekylldev:4000` and see the site.

### Front Matter

Front matter in Jekyll allows you to minimize the amount of code you have to write and rewrite when creating your pages.  Your pages are defined by templates, which is managed with the `_config.yml` file.  In it, you'll be able to enter site wide config settings, along with author based information.  It also allows you to create template based configurations for whatever type of page you choose.  In our example, we're going to focus on the 'post'.

{% highligh yaml %}
\# Defaults
defaults:
  # \_posts
  - scope:
      path: ""
      type: posts             # the pages that this type will affect live in the '\_posts' folder, hence the type: posts.
    values:
      layout: single          # the 'single' layout is located in '\_layouts'
      author_profile: true    # enables or disables the author profile in the side bar
      read_time: true         # estimates for the reader how long it will take to read a post
      comments: true          # enables commenting for the post
      share: true             # allows the reader to share the post
      related: true           # gathers other related articles based on tags
{% endhighlight %}

For more information on layouts for this template, visit the [Minimal Mistakes quick start guide](https://mmistakes.github.io/minimal-mistakes/docs/layouts/).

When you want to make a unique change to the post, you'll want to do that within the front matter.  Michael Rose has a great number of sample posts that you can look at to see what you can do with the front matter.  [Jekyll](https://jekyllrb.com/docs/frontmatter/) has a resource that demonstrates how to customize front matter for your posts.

### Making Your Changes

wnen you've made your changes, you'll want to serve it up to see if it's working right locally, then check it in if your happy with it:

{% highlight bash %}
jekyll serve --host=jekylldev --config=\_config.yml,\_config.dev.yml
git add .
git commit -m 'I am happy with my changes and the site looks right'
git push origin working-branch
{% endhighlight %}

### Workflows

If at all possible, make your changes on the same machine as your dev environment.  If you're running a windows machine, it may be tempting to make your changes using a text editor locally, and then push them up to your dev environment for testing.  While this is doable, it's asking for trouble.  Ensure that you're running CentOS desktop, and install your preferred text editor there.  Make your changes there, and push them up when you're happy with them.  If you want to add pictures, you can add them from your workstation or share them over the network.

Either way, jekyll is way more powerful than any other offering, and can give you a rich user experience for your gitpage.
