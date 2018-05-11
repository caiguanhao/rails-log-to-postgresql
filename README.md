# rails-log-to-postgresql

Parse Rails production log and put it into PostgreSQL.

## 1. Lograge

First, you need to use [Lograge](https://github.com/roidrage/lograge) in your Rails project.

```ruby
# config/environments/production.rb

Rails.application.configure do
  ...
  config.log_level = Rails.env.production? ? :info : :debug
  config.log_tags = [ :subdomain, :remote_ip ]

  config.log_formatter = Class.new(::Logger::Formatter) do
    def call(severity, time, progname, msg)
      "#{severity[0..0]} #{time.strftime('%FT%T')} ##{$$} #{msg2str(msg)}\n"
    end
  end.new

  config.lograge.enabled = true
  config.lograge.base_controller_class = [ 'ActionController::API', 'ActionController::Base' ]
  config.lograge.formatter = Lograge::Formatters::Json.new
  config.lograge.custom_options = lambda do |event|
    { path: event.payload[:path] }
  end
  ...
end
```

Then, your log looks like:

```
I 2018-05-10T00:00:13 #23700 [www] [23.106.203.90] {"method":"GET","path":"/","format":"html","controller":"HomeController","action":"index","status":200,"duration":21.29,"view":19.21,"db":0.0}
```

## 2. lograte

It's better to use lograte to split the log file by date. Example:

```
/my-rails-project/current/log/production.log {
  daily
  missingok
  rotate 99999
  notifempty
  copytruncate
  sharedscripts
  postrotate
    cd /my-rails-project/current/log
    dir=$(date --date=yesterday '+archives/%Y-%m/%d')
    su deploy -c "mkdir -p $dir"
    for f in $(find . -maxdepth 1 -type f -regex '.*\.log\.[0-9]+'); do mv $f $dir/${f%.*}; done
    command -v pigz >/dev/null 2>&1 && gzip=pigz || gzip=gzip
    find $dir -type f -not -name '*.gz' | xargs $gzip -f
  endscript
}
```

To have logrotate run at 00:00:

```
# add this to /etc/crontab and mv /etc/cron.daily/logrotate /etc/cron.daily.really/logrotate
0  0    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily.really )
```
