Clermont'ech Workshop Git
=========================


## Deploying with Git

### 1. Deploy using push/hook

We need two separate folders to simulate two different servers:

    mkdir {local,remote}-server

In the first folder, let's create an app:

    cd local-server
    mkdir app
    git init
    echo "Hello, World" > index.html
    git commit -m "Initial commit"

Let's add a new remote that points to the `production` server:

    git remote add -t master production /absolute/path/to/remote-server/www

Now, we are going to configure the remote server.

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

    if [ "$branch"  == "master" ] ; then
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

This hook is all you need to perform steps at "deploy time".

Let's test it!

    cd ../local-server
    git push production

Enjoy.

### 2. Deploy with Dokku/Heroku

### 3. Capistrano
