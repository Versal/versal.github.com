Versal Engineering Blog Setup
=============================

The Versal Engineering blog is powered by [http://octopress.org](Octopress).

Octopress is written in Ruby.  (Well.)

We strongly recommend using [https://rvm.io](RVM) for compartmentalizing Ruby gem setup.  Install RVM and create a gemset with

`rvm gemset create octopress`

`rvm gemset use octopress`

Now, if you do

`rvm gemset list`

-- you should see `=> octopress` in the list.

Clone the Versal GitHub pages project

`git clone git@github.com:Versal/versal.github.com`

Checkout the `source` branch:

`git checkout -t origin/source`

Now `cd` there and say `yes` to the question whether you trust the `.rvmrc` file -- the local RVM configuration.  The first time you use the `octopress` gemset it will be empty, so you'll need to install a bunch of gems for the bloggin engine, as follows:

     gem install bundler
     bundler install

Create and edit posts in the `source/_posts` directory using your favorite ASCII text editor such as vim or Sublime.

Name the posts as 

     YYYY-MM-DD-url-name-slug-you-will-see-in-them-browsers.md

Start a preview server with

     rake preview

-- and watch the blog preview at `http://localhost:4000`

Once you're satisfied with the results, deploy to GitHub with

     rake deploy

It will take down the existing 
