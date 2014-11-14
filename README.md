Clermont'ech Workshop Git
=========================

The aim of this (very short) workshop is to learn various ways to deploy with
Git. We are going to cover:

* [Deploy Using Git Push/Hook](#1-deploy-using-git-pushhook)
* [Deploy With Heroku](#2-deploy-with-heroku)
* [Deploy With Dokku](#3-deploy-with-dokku)
* [Capistrano](#4-capistrano)


## 1. Deploy Using Git Push/Hook

We need two separate folders to simulate two different servers:

    mkdir {local,remote}-server

In the first folder, let's create an app:

    cd local-server
    mkdir app
    cd !$
    git init
    echo "Hello, World" > index.html
    git add index.html
    git commit -m "Initial commit"

Let's add a new remote that points to the `production` server:

    git remote add production /absolute/path/to/remote-server/www

Now, we are going to configure the remote server.

    cd ../../remote-server
    mkdir www
    cd !$
    git init

Let's edit the `.git/config` file, adding the content below to allow push on the
current branch:

```
[receive]
    denyCurrentBranch = false
```

Now, let's create a `post-receive` hook with the following content:

``` bash
#!/usr/bin/env bash

LOGFILE="/tmp/deploy-app.log"

function notify()
{
    echo "Hi! I am your Git deployer, and I am deploying branch \"$1\" right now :-)"
}

function log()
{
    if [ ! -f "$LOGFILE" ] ; then
        touch "$LOGFILE"
    fi

    echo "$2 ($3) successfully deploy branch: $1" >> "$LOGFILE"
}

while read oldrev newrev ref
do
    branch=`echo $ref | cut -d/ -f3`

    if [ "$branch" ] ; then
        notify "$branch"

        cd ..
        env -i git checkout $branch &> /dev/null
        env -i git reset --hard &> /dev/null

        author_name=`env -i git log -1 --format=format:%an HEAD`
        author_email=`env -i git log -1 --format=format:%ae HEAD`

        log "$branch" "$author_name" "$author_email"
    fi
done
```

Make it executable:

    chmod +x .git/hooks/post-receive

This hook is all you need to perform steps at "deploy time".

Let's test it!

    cd ../../local-server/app
    git push production master

Enjoy:

    ls -l /absolute/path/to/remote-server/www
    tail /tmp/deploy-app.log

**Note:** [git-deploy](https://github.com/mislav/git-deploy) (almost) does the
same thing, but probably better.

### More on this hook

One may want to send emails to the person who just deployed. That is why
`author_name` and `author_email` are retrieved.

Let's try to change the author's email:

    echo "<h1>Hello, World</h1>" > index.html
    git add !$
    GIT_AUTHOR_EMAIL=foo@example.org git commit -m "Surround title with html tags"
    git push production master

### Deploying to another branch

We can deploy another branch:

    git checkout -b feat-content
    echo "<p>content</p>" >> index.html
    git add !$
    git commit -m "add content"
    git push production feat

Result:

    tail /tmp/deploy-app.log
    cat /absolute/path/to/remote-server/www/index.html


## 2. Deploy With Heroku

We are going to deploy a [Node.JS](http://nodejs.org/) application, that is a
[Chat Example](https://github.com/hunterloftis/chat-example).

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/hunterloftis/chat-example)

### Heroku Toolbelt

Install the [Heroku Toolbelt](https://toolbelt.heroku.com/).

Then:

    heroku login
    heroku git:clone -a <heroku-app-id>
    cd !$

Make changes, then deploy:

    git add .
    git commit -m "Make it better"
    git push heroku master


## 3. Deploy With Dokku

[Dokku](https://github.com/progrium/dokku), Docker powered mini-Heroku in around
100 lines of Bash.

Clone the Chat Example:

    git clone git://github.com/hunterloftis/chat-example.git

Deploy it with Dokku: [How To Use the Dokku One-Click Install Image to Deploy
your
App](https://www.digitalocean.com/community/tutorials/how-to-use-the-dokku-one-click-install-image-to-deploy-your-app).

Configure SSH to use the provided keys:

```
Host dokku.clermontech.org
Hostname 178.62.80.95
User dokku
PreferredAuthentications publickey
IdentityFile    ~/.ssh/clermontech_git_rsa
```

Now, let's add a new remote for the Dokku server:

    git remote add deploy dokku@dokku.clermontech.org:your-app-name

In this case, `your-app-name` is my application name. Put whatever you want
here.

Deploy:

    git push deploy master

Browse `http://<your-app-name>.dokku.clermontech.org`.


## 4. Capistrano

[Capistrano](http://capistranorb.com/), a remote server automation and
deployment tool written in Ruby.

Install it:

    gem install capistrano

Then, capify your application:

    cd chat-example
    cap install

Edit `config/deploy.rb`:

``` diff
-set :application, 'my_app_name'
-set :repo_url, 'git@example.com:me/my_repo.git'
+set :application, 'chat-example'
+set :repo_url, 'file:///absolute/path/to/chat-example/.git'

-# set :deploy_to, '/var/www/my_app'
+set :deploy_to, "/absolute/path/to/remote-server/capwww"
```

Edit `config/deploy/production.rb`:

``` diff
-role :app, %w{deploy@example.com}
-role :web, %w{deploy@example.com}
-role :db,  %w{deploy@example.com}
+role :app, %w{username@localhost}
+role :web, %w{username@localhost}

-server 'example.com', user: 'deploy', roles: %w{web app}, my_property: :my_value
+server 'localhost', user: 'username', roles: %w{web app}

-#  set :ssh_options, {
-#    keys: %w(/home/rlisowski/.ssh/id_rsa),
-#    forward_agent: false,
-#    auth_methods: %w(password)
-#  }
+set :ssh_options, {
+  keys: %w(/absolute/path/to/your/.ssh/identity-file),
+  forward_agent: false,
+  auth_methods: %w(publickey)
+}
```

Check your settings:

    cap production deploy:check

Deploy:

    cap production deploy

Verify:

    ls -l /path/to/remote-server/capwww


### Help

Need help with all these Capistrano tasks:

    cap -T

### Manually Start The Application

Manually start the application:

    cd /path/to/remote-server/capwww/current
    npm start

Doh! Bad luck, it does not work.

Let's create a Capistrano task to install NPM dependencies:

``` ruby
namespace :nodejs do

  desc "Install modules non-globally"
  task :npm_install do
    on roles(:app) do
      execute "cd #{current_path} && /usr/bin/env npm install"
    end
  end

end
```

Tell Capistrano to execute it after each deploy:

``` ruby
namespace :deploy do
  # ...

  before :restart, 'nodejs:npm_install'
end
```

Deploy again.

### Automatically Start The Application

Let's add a task to automatically start the application:

``` ruby
namespace :nodejs do
  # ...

  desc 'Start app'
  task :start do
    on roles(:app) do
      execute "cd #{current_path} && /usr/bin/env node index.js &"
    end
  end

end
```

Start your application:

    cap production nodejs:start

