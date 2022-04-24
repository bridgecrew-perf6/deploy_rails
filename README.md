Step 1: install environments (ubuntu)

```sudo apt install curl
curl -sL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

sudo apt-get update
sudo apt-get install git-core zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev nodejs yarn

Install rbenv

git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
exec $SHELL
(Choice ruby version 3.0.2)
rbenv install 3.0.2
rbenv global 3.0.2
ruby -v

gem install bundler

Configuring Git:

git config --global color.ui true
git config --global user.name "YOUR NAME"
git config --global user.email "YOUR@EMAIL.com"
ssh-keygen -t rsa -b 4096 -C "YOUR@EMAIL.com"

gem install rails -v 6.1.4

sudo apt-get install mysql-server mysql-client libmysqlclient-dev
```
Step 2: Install Capistrano
```
  Add gem to develop
  group :development do
    gem 'capistrano'
    gem 'capistrano3-puma'
    gem 'capistrano-bundler'
    gem 'capistrano-rails'
    gem 'capistrano-rbenv'
  end
  
  install cap: cap install
  
In the file deploy.rb
lock '~> 3.17.0'
set :rbenv_ruby, '3.0.2'
set :repo_url, 'link git project'

set :migration_role, 'db'
set :workers, '*': 2
set :log_level, :debug
set :rails_env, 'staging' # invironment use

# Use git remote cache
set :deploy_via, :remote_cache

# Default value for keep_releases is 5
set :keep_releases, 5

# Skip migration if files in db/migrate were not modified
set :conditionally_migrate, true

set :bundle_without, %i[development test]

set :yarn_flags, '--production --check-files'

# Default value for :linked_files is []
set :linked_files, %w[config/master.key config/credentials.yml.enc config/database.yml config/application.yml config/settings.yml]

# Default value for linked_dirs is []
set :linked_dirs, %w[log tmp/pids tmp/cache tmp/sockets vendor/bundle public/uploads public/products node_modules]

before 'deploy:migrate', 'deploy:db_create'
after 'deploy:publishing', 'deploy:restart'

namespace :deploy do
  desc 'Create database'
  task :db_create do
    on roles(:db) do
      with rails_env: fetch(:rails_env) do
        within release_path do
          execute :bundle, :exec, :rake, 'db:create'
        end
      end
    end
  end

  desc 'Seed database'
  task :db_seed do
    on roles(:db) do
      with rails_env: fetch(:rails_env) do
        within release_path do
          execute :bundle, :exec, :rake, 'db:seed'
        end
      end
    end
  end

  desc 'migarte'
  task :migrate do
    on roles(:db) do
      with rails_env: fetch(:rails_env) do
        within release_path do
          execute :bundle, :exec, :rake, 'db:migrate'
        end
      end
    end
  end
end

In capfile.rb 

equire "capistrano/setup"

require "capistrano/deploy"
require "capistrano/scm/git"
install_plugin Capistrano::SCM::Git

require "capistrano/rbenv"
require "capistrano/bundler"
require "capistrano/rails/assets"
require "capistrano/rails/migrations"
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }

In Staging.rb 

set :branch, 'main' # name branch

server 'ip', user: 'root', roles: %w[app web db]

set :application, 'one_page_serve'
set :deploy_to, '/var/www/one_page_server'

```
Step 3: Create puma 
Create file puma.service
touch /etc/systemd/system/puma.service
After edit file below 
```
[Unit]
Description=Puma HTTP Server
After=network.target

# Uncomment for socket activation (see below)
# Requires=puma.socket

[Service]
# Foreground process (do not use --daemon in ExecStart or config.rb)
Type=simple

# Preferably configure a non-privileged user
User=root

# The path to the your application code root directory.
# Also replace the "<YOUR_APP_PATH>" place holders below with this path.
# Example /home/username/myapp
WorkingDirectory=/var/www/one_page_server/current

# Helpful for debugging socket activation, etc.
# Environment=PUMA_DEBUG=1

# SystemD will not run puma even if it is in your path. You must specify
# an absolute URL to puma. For example /usr/local/bin/puma
# Alternatively, create a binstub with `bundle binstubs puma --path ./sbin` in the WorkingDirectory
Environment=RAILS_ENV=staging
ExecStart=/root/.rbenv/bin/rbenv exec bundle exec puma -C /var/www/one_page_server/shared/puma.rb
ExecReload=/bin/kill -TSTP $MAINPID
#StandardOutput=append:/var/www/one_page_serve/current/log/puma.access.log
#StandardError=append:/var/www/one_page_serve/current/log/puma.error.log

Restart=always
RestartSec=1
SyslogIdentifier=puma

[Install]
WantedBy=multi-user.target
```
Create folder project: /var/www/one_page/one_page_server
- In one_page_server: create folder: shared
 Add file config as: config/master.key config/credentials.yml.enc config/database.yml config/application.yml config/settings.yml
Create file puma.rb and edit
```
#!/usr/bin/env puma

directory '/var/www/one_page_server/current'
rackup "/var/www/one_page_server/current/config.ru"
environment 'staging'

tag ''


pidfile "/var/www/one_page_server/shared/tmp/pids/puma.pid"
state_path "/var/www/one_page_server/shared/tmp/pids/puma.state"
stdout_redirect '/var/www/one_page_server/shared/log/puma_access.log', '/var/www/one_page_server/shared/log/puma_error.log', true


threads 0,4



bind 'unix:/var/www/one_page_server/shared/tmp/sockets/puma.sock'

workers 0

restart_command 'bundle exec puma'



prune_bundler


on_restart do
  puts 'Refreshing Gemfile'
  ENV["BUNDLE_GEMFILE"] = ""
end

```

