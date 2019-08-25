## High level overview

Interactive version available at https://www.ixmaps.ca/documentation.php
![IXmaps stack overview](./assets/imgs/stack-overview.png)

### IXmaps backend
https://github.com/ixmaps/php-backend

https://github.com/ixmaps/ixmaps-bin

### IXmaps frontend
https://github.com/ixmaps/website2017

### IXmapsClient
https://github.com/ixmaps/IXmapsClient

### Server setup
#### Website setup
```
git clone https://github.com/ixmaps/website2017.git
cd /var/www/ixmaps
cp config.example.json config.json
less config.json (add gmaps key, modify php-backend as required)

ln -s /var/www/php-backend/application/ application/


git clone https://github.com/ixmaps/trsets.git
# cp -R /var/www/ixmaps-old/IXmapsClient - remove?
# but check git/config for [submodule "trsets"]
#        url = https://github.com/ixmaps/trsets

npm install
bower install
grunt
```

#### Future site of piwik setup
```
--- PIWIK STUFF ---
# cp -R /var/www/ixmaps-old/piwik/ (where does this come from, permissions issues, how can we pull this out so that is not required locally?)
chmod -R www-data piwik
chgrp -R www-data piwik
```

#### PHP setup
```
git clone git@github.com:ixmaps/php-backend.git
cd php-backend/application
cp config.sample.php config.php
nano config.json (add dppassword, modify webUrl if necessary)
ln -s /var/www/php-backend/application/ application/ ??
php-util ??
```

#### Script setup
```
git clone git@github.com:ixmaps/cgi-bin.git /var/www/cgi-bin/
git clone git@github.com:ixmaps/ixmaps-bin.git /home/ixmaps/bin/
```

#### Maxmind data setup?
```
mkdir /home/ixmaps/ix-data/mm-data
python /home/ixmaps/bin/download_maxmind.py ??
```

#### Crontab setup
User ixmaps
```
# download new Maxmind data file (at 2:00 on the 15th of each month)
0 2 15 * * /home/ixmaps/bin/download_maxmind.py

# geocorrection of any new IPs (every 10 minutes)
*/10 * * * * /home/ixmaps/bin/corr-latlong.sh -n
# update cities to match lat and long (every 20 minutes)
*/20 * * * * php /var/www/php-backend/application/controller/geo_update_cities.php > /home/ixmaps/tmp/geo_update_cities.log
# recheck all geocorrection of IPs (every day at 5:00)
0 5 * * * /home/ixmaps/bin/corr-latlong.sh -u

# sql dump to backup ixmaps database (every day at 4:35)
35 4 * * * pg_dump ixmaps | gzip > /home/ixmaps/backup_daily/`date +\%Y\%m\%d`.sql.gz
# long term backup of one sql dump (every Thursday at 4:21)
21 4 * * 4 /bin/cp /home/ixmaps/backup_daily/*01.sql.gz /home/ixmaps/backup
# long term backup of another sql dump (every Thursday at 4:21)
21 4 * * 4 /bin/cp /home/ixmaps/backup_daily/*15.sql.gz /home/ixmaps/backup
# cleanup of daily backups (every Thursday at 4:22)
22 4 * * 4 /usr/bin/find /home/ixmaps/backup_daily -type f -mtime +30 -delete

# concat all of the trsets into 00:_all_trsets.trset (every day at 5:30)
30 5 * * * /home/ixmaps/bin/concat-trsets.sh

# create the helper tables for the DB (every day at 6:00)
0 6 * * * /home/ixmaps/bin/create-extra-db-tables.sh

# collect last hop in tr_last_hops table (100 TR every 20 mins)
# I think this is outdated, waiting for Anto to comment
5,25,40 * * * * php /var/www/php-backend/application/controller/collectLastHop.php > /home/ixmaps/tmp/collectLastHop_log

# update database stats for website
*/5 * * * * /home/ixmaps/bin/db-stats.sh
```
User root
```
# autorenewl of SSL certs (every Monday at 1:00 and 1:05)
0 1 * * 1 /opt/letsencrypt/certbot-auto renew >> /var/log/le-renew.log
5 1 * * 1 service apache2 reload
```
