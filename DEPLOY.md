# Deploy

- These instructions will help you deploy Autolab and Tango in nginx. I found the
  [recommended production deployment
  instructions](https://github.com/autolab/Autolab/wiki/Deploying-Autolab-with-Docker)
  were incomplete, had issues([#1](https://github.com/autolab/Autolab/issues/560),
  [#2](https://github.com/autolab/Autolab/issues/579)) which haven't been resolved
  yet and sometimes didn't work(Tango's Docker-in-Docker for instance). Thus, I
  decided to [expand existing Ubuntu deployment
  instructions](https://github.com/autolab/Autolab/wiki/Deploying-Autolab-on-Ubuntu)
  by adding more details.
- These instructions were tested on Ubuntu 16.04.1 server edition. These should work
  fine on Ubuntu 14.04 too except you might want to use a newer version of
  redis-server than the one present in Ubuntu repositories. I have not tested it on
  Ubuntu 14.04 server though.
- These could serve as a good starting point for other OS too.
- I tested these with Autolab commit
  [4d9e072](https://github.com/autolab/Autolab/commit/4d9e072) and Tango commit
  [db7ad9](https://github.com/autolab/Tango/commit/db7ad9).

## Autolab

- For easy install, keep a sudo user shell and `www-data` shell running simultaneously.
- Reset `www-data`'s password and shell for ease now.
- Create /var/www and set it's owner + group to `www-data`.
- Install packages.
```bash
sudo apt-get install git mysql-server libmysqlclient-dev libsqlite3-dev
```
- Login as `www-data`.
- Clone Autolab source
```bash
git clone https://github.com/autolab/Autolab
```
- Install rbenv + ruby-build
```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
```
- From rbenv README,
	Run `~/.rbenv/bin/rbenv init` for shell-specific instructions on how to
    initialize rbenv to enable shims and autocompletion
- Optionally set `RUBY_BUILD_CACHE_PATH` in `bash_profile` to cache source archives
  downloaded. Useful in case build fails for some reason.
- Optionally set `TMPDIR` and `RUBY_BUILD_BUILD_PATH` in `bash_profile`. `TMPDIR` used
  for temporary files. `RUBY_BUILD_BUILD_PATH` is build directory and sub-directory of
  `TMPDIR`.
- Logout and login as `www-data`.
- Install ruby build dependencies as sudo user.
```bash
sudo apt-get build-dep ruby2.3
```
- Install ruby from Autolab folder as `www-data`.
```bash
rbenv install $(cat .ruby-version)
```
- Check if ruby and rake are detected correctly as `www-data`.
```bash
which ruby
which rake
```
- Install dependencies using bundler from the Autolab directory as `www-data`.
```bash
gem install bundler
rbenv rehash
bundle install
```
- While bundle installs everything, configure database as `www-data`. Copy template
  config.
```bash
cp config/database.yml.template config/database.yml
```
- Create unprivileged user inside MySQL by logging is as root.
```mysql
mysql> CREATE USER app IDENTIFIED BY '[insert password]';
```
- Change user, password and database fields in config/database.yml. Also, create
  section 'production' with identical values as deployment.
- Configure Devise.
```bash
cp config/initializers/devise.rb.template config/initializers/devise.rb
bundle exec rake secret
```
  Copy above output to secret_key in devise.rb.
- Create and migrate DB after bundle has completed installing.
```bash
bundle exec rake db:create
bundle exec rake db:migrate
```
- Run dev server to ensure everything is working.
```bash
bundle exec rails s -p 3000 -b [server]
```
- Install nginx with passenger support. See
  https://www.phusionpassenger.com/library/install/nginx/install/oss/xenial/. This
  is better because the other alternative is to compile nginx from source since nginx
  doesn't support dynamic loading of plugins.
- Copy docker/nginx.template.conf to /etc/nginx/sites-available.
```bash
sudo cp /var/www/Autolab/docker/nginx.template.conf /etc/nginx/sites-available/autolab.conf
```
- Modify the following fields: server_name, passenger_user and passenger_ruby. Set
  passenger_ruby to path returned by `which ruby` when run by `www-data`(probably
  /var/www/.rbenv/shims/ruby). Enable SSL if needed.
- Set root field to `/var/www/Autolab/public`.
- Symlink autolab.conf file to /etc/nginx/sites-enabled/.
```bash
sudo ln -s /etc/nginx/sites-available/autolab.conf /etc/nginx/sites-enabled/autolab.conf
```
- Run following command from Autolab folder as `www-data` to collect all static files.
```bash
RAILS_ENV=production rake assets:precompile
```
- Restart nginx after above command exits.
- Navigate to http://[server] to view Autolab running.

## Autolab production configuration setup

There are 3 key tasks:

1. Configuring email sending.
2. Configuring email sending when an exception occurs.
3. Enabling/disabling HTTPS.

Of these, configuration steps for [sending email when an exception
occurs](https://github.com/autolab/Autolab/wiki/Deploying-Autolab-with-Docker#72-configure-exception-notifications)
and [enabling/disabling HTTPS](https://github.com/autolab/Autolab/wiki/Deploying-Autolab-with-Docker#8-https)
are well documented in the Autolab wiki and best read there. Here are steps for
setting up exim4 as a send-only mail server for sending notifications.

- Install and configure exim4 as described in [this
  guide](https://www.linode.com/docs/email/exim/deploy-exim-as-a-send-only-mail-server-on-ubuntu-12-04).
- After testing if exim4 is able to send emails correctly, configure an email ID to
  use as sender email ID in /etc/email-addresses. This will automatically read by
  exim4 before sending email. Make sure you specify the username as the one Autolab
  runs as.
- Configure SMTP details in production.rb as [described
  here](https://github.com/autolab/Autolab/wiki/Deploying-Autolab-with-Docker#71-configure-email-with-mandrill).
  Also, set the mailer_send configuration in `Autolab/config/initalizers/devise.rb`.
- Restart exim4 and nginx. Email notification should be configured and working
  correctly in Autolab.

## Tango

- Install pip redis-server supervisor.
```bash
sudo apt-get install python-pip redis-server supervisor
```
- Install docker. See https://docs.docker.com/engine/installation/.
- Obtain Tango source code as `www-data`.
```bash
git clone https://github.com/autolab/Tango.git; cd Tango
```
- Build autograding image.
```bash
docker build -t autograding_image vmms/
```
- Install Tango requirements in a virtualenv as `www-data` from Tango source tree.
```bash
sudo pip install virtualenv
virtualenv .
source bin/activate
pip install -r requirements.txt
```
- Create Tango config as `www-data` from template by modifying following PREFIX,
  COURSELABS, DOCKER_VOLUME_PATH and USE_REDIS.
- Change Tango's restful server(`Tango/restful-tango/server.py`) to listen only on
  localhost if you are running Tango on same machine as Autolab. The Tango processes
  run with root privileges since they have to interact with Docker daemon and thus,
  exposing them outside of localhost is a security risk and best avoided if
  unnecessary. Additionally, nginx acts as a proxy for Tango so there's no need to
  expose Tango. Change the listen directive in the server file to something like:
```python
application.listen(port, address='127.0.0.1', max_buffer_size=Config.MAX_INPUT_FILE_SIZE)
```
- Create the COURSELABS and DOCKER_VOLUME_PATH directories exist as `www-data`.
- Copy sections [program:tango] and [program:tangoJobManager] from
  deployment/config/supervisord.conf to /etc/supervisor/conf.d/tango.conf.
- Modify the path to the Tango directory specified in tango.conf.
- Change command in both sections to just the `python` commands. This ensures that
  supervisor will kill the Tango processes before restarting. Otherwise, the old
  processes are not killed before restarting. Alternatively, use `stopasgroup` and/or
  `killasgroup` configuration directives of supervisord. I have not tested these.
- Add following lines to both sections in tango.conf. These ensure that the virtualenv
  created above is used when running Tango and the processes automatically restart
  when they exit.
```
environment=PATH="/var/www/Tango/bin:%(ENV_PATH)s"
autorestart=True
```
- Somehow supervisor wasn't added to system startup. Do so using `update-rc.d`.
```bash
sudo update-rc.d supervisor enable
```
- After starting the Tango service using supervisor, navigate to [server]:8610 and
  [server]:8611 to ensure Tango is running correctly.
- Copy the server and upstream sections deployment/config/nginx.conf to
  /etc/nginx/sites-available/tango.conf. Symlink this file to
  /etc/nginx/sites-enabled/. Change the listen directive to make nginx listen for
  connections over localhost if running on same machine as Autolab.
- Restart nginx. Navigate to http://[server]:8600 to view Tango's message.
- Set RESTFUL_HOST, RESTFUL_PORT and RESTFUL_KEY in
  Autolab/config/autogradeConfig.rb. Use template file provided if file doesn't
  exist.
- Restart nginx again.
- Lock `www-data` account and unset `www-data`'s shell.
