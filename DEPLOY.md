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

## Autolab

- For easy install, keep a sudo user shell and www-data shell running simultaneously.
- Reset www-data's password and shell for ease now.
- Create /var/www and set it's owner + group to www-data.
- Install packages: git mysql-server libmysqlclient-dev libsqlite3-dev.
- Login as www-data.
- Clone Autolab source: git clone https://github.com/autolab/Autolab.
- Install rbenv + ruby-build.
    - git clone https://github.com/rbenv/rbenv.git ~/.rbenv
	- git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
	- From rbenv README,
		Run ~/.rbenv/bin/rbenv init for shell-specific instructions on how to
        initialize rbenv to enable shims and autocompletion.
    - $ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
	- Optionally set RUBY_BUILD_CACHE_PATH in bash_profile to cache source archives
      downloaded. Useful in case build fails for some reason.
	- Optionally set TMPDIR and RUBY_BUILD_BUILD_PATH in bash_profile. TMPDIR used
      for temporary files. RUBY_BUILD_BUILD_PATH is build directory and sub-directory
      of TMPDIR.
    - Logout and login as www-data.
- Install ruby build dependencies as sudo user.
    - $ sudo apt-get build-dep ruby2.3.
- Install ruby from Autolab folder as www-data.
    - $ rbenv install $(cat .ruby-version).
- Check if ruby and rake are detected correctly as www-data.
    - $ which ruby.
    - $ which rake.
- Install dependencies using bundler from the Autolab directory as www-data.
    - $ gem install bundler
    - $ rbenv rehash
    - $ bundle install
- While bundle installs everything, configure database as www-data. Copy template
  config.
    - $ cp config/database.yml.template config/database.yml
- Create unprivileged user inside MySQL by logging is as root.
    - mysql> CREATE USER app IDENTIFIED BY '<insert password>';
- Change user, password and database fields in config/database.yml. Also, create
  section 'production' with identical values as deployment.
- Configure Devise.
    - $ cp config/initializers/devise.rb.template config/initializers/devise.rb
    - $ bundle exec rake secret
    - Copy above output to secret_key in devise.rb.
- Create and migrate DB after bundle has completed installing.
    - $ bundle exec rake db:create
    - $ bundle exec rake db:migrate
- Run dev server to ensure everything is working.
    - $ bundle exec rails s -p 3000 -b <server>
- Install nginx with passenger support. See
  https://www.phusionpassenger.com/library/install/nginx/install/oss/xenial/. This
  is better because the other alternative is to compile nginx from source since nginx
  doesn't support dynamic loading of plugins.
- Copy docker/nginx.template.conf to /etc/nginx/sites-available.
    - sudo cp /var/www/Autolab/docker/nginx.template.conf /etc/nginx/sites-available/autolab.conf
- Modify the following fields: server_name, passenger_user and passenger_ruby. Set
  passenger_ruby to path returned by `which ruby` when run by www-data(probably
  /var/www/.rbenv/shims/ruby). Enable SSL if needed.
- Symlink autolab.conf file to /etc/nginx/sites-enabled/.
    - sudo ln -s /etc/nginx/sites-available/autolab.conf /etc/nginx/sites-enabled/autolab.conf
- Run following command from Autolab folder as www-data to collect all static files.
    - $ RAILS_ENV=production rake assets:precompile
- Restart nginx after above command exits.
- Navigate to http://<server> to view Autolab running.

## Tango

- Install pip redis-server supervisor.
    - $ sudo apt-get install python-pip redis-server supervisor
- Install docker. See https://docs.docker.com/engine/installation/.
- Obtain Tango source code as www-data.
    - $ git clone https://github.com/autolab/Tango.git; cd Tango
- Build autograding image.
    - $ docker build -t autograding_image vmms/
- Install Tango requirements in a virtualenv as www-data from Tango source tree.
    - $ sudo pip install virtualenv
    - $ virtualenv .
    - $ source bin/activate
    - $ pip install -r requirements.txt
- Create Tango config as www-data from template by modifying following PREFIX,
  COURSELABS, DOCKER_VOLUME_PATH and USE_REDIS.
- Create the COURSELABS and DOCKER_VOLUME_PATH directories exist as www-data.
- Copy sections [program:tango] and [program:tangoJobManager] from
  deployment/config/supervisord.conf to /etc/supervisor/conf.d/tango.conf.
- Modify the path to the Tango directory specified in tango.conf.
- Add following line to both sections in tango.conf. This ensures the virtualenv
  created above is used when running Tango.
  - environment=PATH="/var/www/Tango/bin:%(ENV_PATH)s"
- Somehow supervisor wasn't added to system startup. Do so using update-rc.d.
    - $ sudo update-rc.d supervisor enable.
- After starting the Tango service using supervisor, navigate to <server>:8610 and
  <server>:8611 to ensure Tango is running correctly.
- Copy the server and upstream sections deployment/config/nginx.conf to
  /etc/nginx/sites-available/tango.conf. Symlink this file to
  /etc/nginx/sites-enabled/.
- Restart nginx. Navigate to <server>:8600 to view Tango's message.
- Set RESTFUL_HOST, RESTFUL_PORT and RESTFUL_KEY in
  Autolab/config/autogradeConfig.rb. Use template file provided if file doesn't
  exist.
- Restart nginx again.