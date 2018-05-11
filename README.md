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

## 3. import

Download the log file `production.log`, and create a database `my_test_db`.

And put following code to `import.sh` and make it executable `chmod +x import.sh`

Run `./import.sh` to start.

```bash
#!/bin/bash

LOGFILE=production.log
DBNAME=my_test_db

echo '
DROP TABLE IF EXISTS weblogstemp;
CREATE TABLE weblogstemp (
  time timestamp without time zone NOT NULL,
  subdomain character varying NOT NULL,
  ip_address inet NOT NULL,
  json json NOT NULL
);
' | psql $DBNAME

awk '
BEGIN {
  FS="\\[|\\]| "
  print "COPY weblogstemp (time, subdomain, ip_address, json) FROM stdin;"
}
/{"method"/ {
  printf("%s", $2"\t"$5"\t"$8"\t");
  $1=$2=$3=$4=$5=$6=$7=$8=$9="";
  print substr($0,10);
}
END {
  print "\\."
}
' $LOGFILE | psql $DBNAME

echo '
DROP TYPE IF EXISTS weblogjson;
CREATE TYPE weblogjson AS (
  method character varying,
  path character varying,
  format character varying,
  controller character varying,
  action character varying,
  status character varying,
  duration numeric(6,2),
  view numeric(6,2),
  db numeric(6,2)
);
DROP TABLE IF EXISTS weblogs;
CREATE TABLE weblogs AS (
  SELECT w.time, w.subdomain, w.ip_address, j.* FROM
  weblogstemp w, json_populate_record(null::weblogjson, json) j
);
DROP TYPE weblogjson;
DROP TABLE weblogstemp;
' | psql $DBNAME
```

It takes about 15 secs to import 500k lines (requests) on an MBP.

## 4. query

You can now run `psql my_test_db` to query the log.

```
        time         | subdomain |  ip_address  | method |   path   | format |   controller    | action | status | duration | view |  db
---------------------+-----------+--------------+--------+----------+--------+-----------------+--------+--------+----------+------+------
 2018-05-10 01:56:44 | www       | 117.75.19.61 | GET    | /users/1 | */*    | UsersController | show   | 200    |    12.54 | 7.63 | 2.63
(1 row)
```

### Count

```
# select count(*) from weblogs;
 count
--------
 553656
(1 row)
```

### Count by controller

```
# select controller, count(*) c from weblogs group by controller order by c desc;
            controller             |   c
-----------------------------------+--------
 PostsController                   | 295102
 UsersController                   |  77635
 SessionsController                |  63666
 HomeController                    |  47266
 ApplicationController             |  31312
...
```

### Count by status

```
# select status, count(*) c from weblogs group by status order by c desc;
 status |   c
--------+--------
 200    | 393259
 302    | 104891
 404    |  46658
 400    |   5540
 422    |   2960
 204    |    305
 500    |     39
 401    |      4
(8 rows)
```

### Count by IP address

```
# select ip_address, count(*) c from weblogs group by ip_address order by c desc;
   ip_address    |   c
-----------------+-------
 23.106.203.90   | 17734
 104.151.73.2    |  8476
 198.16.59.178   |  7824
 142.234.255.76  |  7016
...
```

### Count by hour

```
# select date_part('hour', time), count(*) c from weblogs where path = '/' group by 1 order by c desc;
 date_part |  c
-----------+------
         9 | 2321
        11 | 2220
        17 | 2017
        10 | 2012
        22 | 1904
        15 | 1857
        16 | 1846
        14 | 1811
        23 | 1778
        18 | 1678
         8 | 1517
        20 | 1512
        21 | 1512
        13 | 1481
         5 | 1426
         0 | 1405
         1 | 1403
        12 | 1396
        19 | 1367
         7 | 1227
         4 | 1036
         3 | 1028
         6 |  970
         2 |  954
(24 rows)
```

### Find the slowest requests

```
# select regexp_replace(path, '\?.*', '') path, max(duration) duration from weblogs group by 1 order by 2 desc limit 5;
   path   | duration
----------+----------
 /ydvgd   |  4464.97
 /geetest |  4095.70
 /        |  2867.22
 /forums  |  2695.80
 /search  |  2601.04
(5 rows)
```
