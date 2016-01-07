# Plugin triggers

[Plugin triggers](https://github.com/dokku/plugn) (formerly [pluginhooks](https://github.com/progrium/pluginhook)) are a good way to jack into existing dokku infrastructure. You can use them to modify the output of various dokku commands or override internal configuration.

Plugin triggers are simply scripts that are executed by the system. You can use any language you want, so long as the script:

- Is executable
- Has the proper language requirements installed

For instance, if you wanted to write a plugin trigger in PHP, you would need to have `php` installed and available on the CLI prior to plugin trigger invocation.

The following is an example for the `nginx-hostname` plugin trigger. It reverses the hostname that is provided to nginx during deploys. If you created an executable file named `nginx-hostname` with the following code in your plugin trigger, it would be invoked by dokku during the normal app deployment process:

```shell
#!/usr/bin/env bash
set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

APP="$1"; SUBDOMAIN="$2"; VHOST="$3"

NEW_SUBDOMAIN=`echo $SUBDOMAIN | rev`
echo "$NEW_SUBDOMAIN.$VHOST"
```

## Available plugin triggers

There are a number of plugin-related triggers. These can be optionally implemented by plugins and allow integration into the standard dokku setup/backup/teardown process.

The following plugin triggers describe those available to a dokku installation. As well, there is an example for each trigger that you can use as templates for your own plugin development.

> The example plugin trigger code is not guaranteed to be implemented as in within dokkku, and are merely simplified examples. Please look at the dokku source for larger, more in-depth examples.

### `install`

- Description: Used to setup any files/configuration for a plugin.
- Invoked by: `dokku plugin:install`.
- Arguments: None
- Example:

```shell
#!/usr/bin/env bash
# Sets the hostname of the dokku server
# based on the output of `hostname -f`

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

if [[ ! -f  "$DOKKU_ROOT/HOSTNAME" ]]; then
  hostname -f > $DOKKU_ROOT/HOSTNAME
fi
```

### `dependencies`

- Description: Used to install system-level dependencies. Invoked by `plugin:install-dependencies`.
- Invoked by: `dokku plugin:install-dependencies`
- Arguments: None
- Example:

```shell
#!/usr/bin/env bash
# Installs nginx for the current plugin
# Supports both opensuse and ubuntu

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

export DEBIAN_FRONTEND=noninteractive

case "$DOKKU_DISTRO" in
  debian|ubuntu)
    apt-get install --force-yes -qq -y nginx
    ;;

  opensuse)
    zypper -q in -y nginx
    ;;
esac
```

### `update`

- Description: Can be used to run plugin updates on a regular interval. You can schedule the invoker in a cron-task to ensure your system gets regular updates.
- Invoked by: `dokku plugin:update`.
- Arguments: None
- Example:

```shell
#!/usr/bin/env bash
# Update the herokuish image from git source

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

cd /root/dokku
sudo BUILD_STACK=true make install
```

### `commands help`

- Description: Used to aggregate all plugin `help` output. Your plugin should implement a `help` command in your `commands` file to take advantage of this plugin trigger. This must be implemented inside the `commands` plugin file.
- Invoked by: `dokku help`
- Arguments: None
- Example:

```shell
#!/usr/bin/env bash
# Outputs help for the derp plugin

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

case "$1" in
  help | derp:help)
    cat<<EOF
    derp:herp, Herps the derp
    derp:serp [file], Shows the file's serp
EOF
    ;;

  *)
    exit $DOKKU_NOT_IMPLEMENTED_EXIT
    ;;

esac
```

### `backup-export`

- Description: Used to backup files for a given plugin. If your plugin writes files to disk, this plugin trigger should be used to echo out their full paths. Any files listed will be copied by the backup plugin to the backup tar.gz.
- Invoked by: `dokku backup:export`
- Arguments: `$VERSION $DOKKU_ROOT`
- Example:

```shell
#!/usr/bin/env bash
# Echos out the location of every `REDIRECT` file
# that are used by the apps

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

shopt -s nullglob
VERSION="$1"
DOKKU_ROOT="$2"

cat; for i in $DOKKU_ROOT/*/REDIRECT; do echo $i; done
```

### `backup-check`

- Description: Checks that a backup being imported passes sanity checks.
- Invoked by: `dokku backup:import`
- Arguments: `$VERSION $BACKUP_ROOT $DOKKU_ROOT $BACKUP_TMP_DIR/.dokku_backup_apps`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `backup-import`

- Description: Allows a plugin to import specific files from a `$BACKUP_ROOT` to the `DOKKU_ROOT` directory.
- Invoked by: `dokku backup:import`
- Arguments: `$VERSION $BACKUP_ROOT $DOKKU_ROOT $BACKUP_TMP_DIR/.dokku_backup_apps`
- Example:

```shell
#!/usr/bin/env bash
# Copies all redirect files from the backup
# into the correct app path.

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

shopt -s nullglob
VERSION="$1"
BACKUP_ROOT="$2"
DOKKU_ROOT="$3"

cd $BACKUP_ROOT
for file in */REDIRECT; do
  cp $file "$DOKKU_ROOT/$file"
done
```

### `pre-build-buildpack`

- Description: Allows you to run commands before the build image is created for a given app. For instance, this can be useful to add env vars to your container. Only applies to applications using buildpacks.
- Invoked by: `dokku build`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `post-build-buildpack`

- Description: Allows you to run commands after the build image is create for a given app. Only applies to applications using buildpacks.
- Invoked by: `dokku build`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `pre-build-dockerfile`

- Description: Allows you to run commands before the build image is created for a given app. For instance, this can be useful to add env vars to your container. Only applies to applications using a dockerfile.
- Invoked by: `dokku build`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `post-build-dockerfile`

- Description: Allows you to run commands after the build image is create for a given app. Only applies to applications using a dockerfile.
- Invoked by: `dokku build`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `pre-release-buildpack`

- Description: Allows you to run commands before environment variables are set for the release step of the deploy. Only applies to applications using buildpacks.
- Invoked by: `dokku release`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Installs the graphicsmagick package into the container

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

dokku_log_info1" Installing GraphicsMagick..."

CMD="cat > gm && \
  dpkg -s graphicsmagick > /dev/null 2>&1 || \
  (apt-get update && apt-get install -y graphicsmagick && apt-get clean)"

ID=$(docker run $DOKKU_GLOBAL_RUN_ARGS -i -a stdin $IMAGE /bin/bash -c "$CMD")
test $(docker wait $ID) -eq 0
docker commit $ID $IMAGE > /dev/null
```

### `post-release-buildpack`

- Description: Allows you to run commands after environment variables are set for the release step of the deploy. Only applies to applications using buildpacks.
- Invoked by: `dokku release`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Installs a package specified by the `CONTAINER_PACKAGE` env var

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

dokku_log_info1" Installing $CONTAINER_PACKAGE..."

CMD="cat > gm && \
  dpkg -s CONTAINER_PACKAGE > /dev/null 2>&1 || \
  (apt-get update && apt-get install -y CONTAINER_PACKAGE && apt-get clean)"

ID=$(docker run $DOKKU_GLOBAL_RUN_ARGS -i -a stdin $IMAGE /bin/bash -c "$CMD")
test $(docker wait $ID) -eq 0
docker commit $ID $IMAGE > /dev/null
```

### `pre-release-dockerfile`

- Description: Allows you to run commands before environment variables are set for the release step of the deploy. Only applies to applications using a dockerfile.
- Invoked by: `dokku release`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

# TODO
```

### `post-release-dockerfile`

- Description: Allows you to run commands after environment variables are set for the release step of the deploy. Only applies to applications using a dockerfile.
- Invoked by: `dokku release`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

# TODO
```

### `check-deploy`

- Description: Allows you to run checks on a deploy before dokku allows the container to handle requests.
- Invoked by: `dokku deploy`
- Arguments: `$CONTAINER_ID $APP $INTERNAL_PORT $INTERNAL_IP_ADDRESS`
- Example:

```shell
#!/usr/bin/env bash
# Disables deploys of containers based on whether the
# `DOKKU_DISABLE_DEPLOY` env var is set to `true` for an app

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_AVAILABLE_PATH/config/functions"

CONTAINERID="$1"; APP="$2"; PORT="$3" ; HOSTNAME="${4:-localhost}"

eval "$(config_export app $APP)"
DOKKU_DISABLE_DEPLOY="${DOKKU_DISABLE_DEPLOY:-false}"

if [[ "$DOKKU_DISABLE_DEPLOY" = "true" ]]; then
  echo -e "\033[31m\033[1mDeploys disabled, sorry.\033[0m"
  exit 1
fi
```

### `pre-deploy`

- Description: Allows the running of code before the container's process is started.
- Invoked by: `dokku deploy`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Runs gulp in our container

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

dokku_log_info1 "Running gulp"
id=$(docker run $DOKKU_GLOBAL_RUN_ARGS -d $IMAGE /bin/bash -c "cd /app && gulp default")
test $(docker wait $id) -eq 0
docker commit $id $IMAGE > /dev/null
dokku_log_info1 "Building UI Complete"
```

### `post-create`

- Description: Can be used to run commands after an application is created.
- Invoked by: `dokku apps:create`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash
# Runs a command to ensure that an app
# has a postgres database when it is starting

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

APP="$1";
POSTGRES="$1"

dokku postgres:create $POSTGRES
dokku postgres:link $POSTGRES $APP
```

### `post-deploy`

- Description: Allows running of commands after a deploy has completed. Dokku core currently uses this to switch traffic on nginx.
- Invoked by: `dokku deploy`
- Arguments: `$APP $INTERNAL_PORT $INTERNAL_IP_ADDRESS $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Notify an external service that a successful deploy has occurred.

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

curl "http://httpstat.us/200"
```

### `pre-delete`

- Description: Can be used to run commands before an app is deleted.
- Invoked by: `dokku apps:destroy`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Clears out the gulp asset build cache for applications

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"

APP="$1"; GULP_CACHE_DIR="$DOKKU_ROOT/$APP/gulp"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

if [[ -d $GULP_CACHE_DIR ]]; then
  docker run $DOKKU_GLOBAL_RUN_ARGS --rm -v "$GULP_CACHE_DIR:/gulp" "$IMAGE" find /gulp -depth -mindepth 1 -maxdepth 1 -exec rm -Rf {} \; || true
fi
```

### `post-delete`

- Description: Can be used to run commands after an application is deleted.
- Invoked by: `dokku apps:destroy`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Runs a command to ensure that an app's
# postgres installation is removed

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

APP="$1";

dokku postgres:destroy $APP
```

### `post-stop`

- Description: Can be used to run commands after an application is manually stopped
- Invoked by: `dokku ps:stop`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash
# Marks an application as manually stopped

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

APP="$1";

dokku config:set --no-restart $APP MANUALLY_STOPPED=1
```

### `docker-args-build`

- Description:
- Invoked by: `dokku build`
- Arguments: `$APP $IMAGE_SOURCE_TYPE`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `docker-args-deploy`

- Description:
- Invoked by: `dokku deploy`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

# TODO
```

### `docker-args-run`

- Description:
- Invoked by: `dokku run`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
source "$PLUGIN_CORE_AVAILABLE_PATH/common/functions"
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)
verify_app_name "$APP"

# TODO
```

### `bind-external-ip`

- Description: Allows you to disable binding to the external box ip
- Invoked by: `dokku deploy`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash
# Force always binding to the docker ip, no matter
# what the settings are for a given app.

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

echo false
```

### `post-domains-update`

- Description: Allows you to run commands once the domain for an application has been updated. It also sends in the command that has been used. This can be "add", "clear" or "remove". The third argument will be the optional list of domains
- Invoked by: `dokku domains:add`, `dokku domains:clear`, `dokku domains:remove`
- Arguments: `$APP` `action name` `domains`
- Example:

```shell
#!/usr/bin/env bash
# Reloads haproxy for our imaginary haproxy plugin
# that replaces the nginx-vhosts plugin

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

sudo service haproxy reload
```

### `git-pre-pull`

- Description:
- Invoked by: `dokku git-upload-pack`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `git-post-pull`

- Description:
- Invoked by: `dokku git-upload-pack`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

# TODO
```

### `nginx-hostname`

- Description: Allows you to customize the hostname for a given application.
- Invoked by: `dokku domains:setup`
- Arguments: `$APP $SUBDOMAIN $VHOST`
- Example:

```shell
#!/usr/bin/env bash
# Reverses the hostname for the application

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

APP="$1"; SUBDOMAIN="$2"; VHOST="$3"

NEW_SUBDOMAIN=`echo $SUBDOMAIN | rev`
echo "$NEW_SUBDOMAIN.$VHOST"
```

### `nginx-pre-reload`

- Description: Run before nginx reloads hosts
- Invoked by: `dokku nginx:build-config`
- Arguments: `$APP $INTERNAL_PORT $INTERNAL_IP_ADDRESS`
- Example:

```shell
#!/usr/bin/env bash
# Runs a check against all nginx conf files
# to ensure they are valid

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

nginx -t
```

### `pre-receive-app`

- Description: Allows you to customize the contents of an application directory before they are processed for deployment. The `IMAGE_SOURCE_TYPE` can be any of `[herokuish, dockerfile]`
- Invoked by: `dokku git-hook`, `dokku tar-build-locked`
- Arguments: `$APP $IMAGE_SOURCE_TYPE $TMP_WORK_DIR $REV`
- Example:

```shell
#!/usr/bin/env bash
# Adds a file called `dokku-is-awesome` to the repository
# the contents will be the application name

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

APP="$1"; IMAGE_SOURCE_TYPE="$2"; TMP_WORK_DIR="$3"; REV="$4"

echo "$APP" > "$TMP_WORK_DIR/dokku-is-awesome"
```

### `receive-app`

- Description: Allows you to customize what occurs when an app is received. Normally just triggers an application build.
- Invoked by: `dokku git-hook`, `dokku ps:rebuild`
- Arguments: `$APP $REV`
- Example:

```shell
#!/usr/bin/env bash
# For our imaginary mercurial plugin, triggers a rebuild

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

APP="$1"; REV="$2"

dokku hg-build $APP $REV
```

### `receive-branch`

- Description: Allows you to customize what occurs when a specific branch is received. Can be used to add support for specific branch names
- Invoked by: `dokku git-hook`, `dokku ps:rebuild`
- Arguments: `$APP $REV $REFNAME`
- Example:

```shell
#!/bin/bash
# Gives dokku the ability to support multiple branches for a given service
# Allowing you to have multiple staging environments on a per-branch basis

reference_app=$1
refname=$3
newrev=$2
APP=${refname/*\//}.$reference_app

if [[ ! -d "$DOKKU_ROOT/$APP" ]]; then
  REFERENCE_REPO="$DOKKU_ROOT/$reference_app
  git clone --bare --shared --reference "$REFERENCE_REPO" "$REFERENCE_REPO" "$DOKKU_ROOT/$APP" > /dev/null
fi
plugn trigger receive-app $APP $newrev
```

### `tags-create`

- Description: Allows you to run commands once a tag for an application image has been added
- Invoked by: `dokku tags:create`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Upload an application image to docker hub

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
APP="$1"; IMAGE_TAG="$2"; IMAGE=$(get_app_image_name $APP $IMAGE_TAG)

IMAGE_ID=$(docker inspect --format '{{ .Id }}' $IMAGE)
docker tag -f $IMAGE_ID $DOCKER_HUB_USER/$APP:$IMAGE_TAG
docker push $DOCKER_HUB_USER/$APP:$IMAGE_TAG
```

### `tags-destroy`

- Description: Allows you to run commands once a tag for an application image has been removed
- Invoked by: `dokku tags:destroy`
- Arguments: `$APP $IMAGE_TAG`
- Example:

```shell
#!/usr/bin/env bash
# Remove an image tag from docker hub

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
APP="$1"; IMAGE_TAG="$2"

some code to remove a docker hub tag because it's not implemented in the CLI....
```

### `retire-container-failed`

- Description: Allows you to run commands if/when retiring old containers has failed
- Invoked by: `dokku deploy`
- Arguments: `$APP`
- Example:

```shell
#!/usr/bin/env bash
# Send an email when a container failed to retire

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
APP="$1"; HOSTNAME=$(hostname -s)

mail -s "$APP containers on $HOSTNAME failed to retire" ops@example.com
```

### `user-auth`

This is a special plugin trigger that is executed on *every* command run. As dokku sometimes internally invokes the `dokku` command, special care should be taken to properly handle internal command redirects.

Note that the trigger should exit as follows:

- `0` to continue running as normal
- `1` to halt execution of the command

The `SSH_USER` is the original ssh user. If you are running remote commands, this user will typically be `dokku`, and as such should not be trusted when checking permissions. If you are connected via ssh as a different user who then invokes `dokku`, the value of this variable will be that user's name (`root`, `myuser`, etc.).

The `SSH_NAME` is the `NAME` variable set via the `sshcommand acl-add` command. If you have set a user via the `dokku-installer`, this value will be set to `admin`. For installs via debian package, this value *may* be `default`. For reference, the following command can be run as the root user to specify a specific `NAME` for a given ssh key:

```shell
sshcommand acl-add dokku NAME < $PATH_TO_SSH_KEY
```

Note that the `NAME` value is set at the first ssh key match. If an ssh key is set in the `/home/dokku/.ssh/authorized_keys` multiple times, the first match will decide the value.

- Description: Allows you to deny access to a dokku command by either ssh user or associated ssh-command NAME user.
- Invoked by `dokku`
- Arguments: `$SSH_USER $SSH_NAME $DOKKU_COMMAND`
- Example:

```shell
#!/usr/bin/env bash
# Allow root/admin users to do everything
# Deny plugin access to default users
# Allow access to all other commands

set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x

SSH_USER=$1
SSH_NAME=$2
shift 2
[[ "$SSH_USER" == "root" ]] && exit 0
[[ "$SSH_NAME" == "admin" ]] && exit 0
[[ "$SSH_NAME" == "default" && $1 == plugin:* ]] && exit 1
exit 0
```
