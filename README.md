Clermont'ech Workshop Git
=========================


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


## 2. Deploy With Dokku/Heroku


## 3. Capistrano
