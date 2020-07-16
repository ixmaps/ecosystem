## High level overview

Interactive version available at https://www.ixmaps.ca/documentation.php
![IXmaps stack overview](./assets/imgs/stack-overview.png)

### IXmaps frontend
https://github.com/ixmaps/website2017
```
git clone https://github.com/ixmaps/website2017.git /srv/www/website
cp /srv/www/website/config.example.json /srv/www/website/config.json
nano config.json (add Google Maps key, possibly update 'php_backend' value)
cd /srv/www/website/_assets/__build
npm install
bower install
```

Follow the setup instructions in the README listed at https://github.com/ixmaps/website2017.git
(todo: think about moving those instructions here? Or there instructions there?)

### IXmaps backend
https://github.com/ixmaps/php-backend
https://github.com/ixmaps/ixmaps-bin

#### PHP backend setup
```
sudo mkdir /srv/www/php-backend
sudo chown ixmaps:ixmaps /srv/www/php-backend
git clone https://github.com/ixmaps/php-backend.git /srv/www/php-backend
cd /srv/www/php-backend/application
cp /srv/www/php-backend/application/config.sample.php /srv/www/php-backend/application/config.php
nano /srv/www/php-backend/application/config.php to add dbpassword (create in db if necessary), modify webUrl if necessary
cp /srv/www/php-backend/application/config.example.json /srv/www/php-backend/application/config.json
nano /srv/www/php-backend/application/config.js (add gmaps key)
ln -s /srv/www/php-backend/application/ /srv/www/website/
```

#### Utility scripts
```
git clone https://github.com/ixmaps/ixmaps-bin.git /home/ixmaps/bin
```

#### Maxmind data setup
```
mkdir /home/ixmaps/ix-data
mkdir /home/ixmaps/ix-data/mm-data
/home/ixmaps/bin/download_maxmind.sh
```

#### Seeding
```
scp ixmaps_seed.sql.gz ixmaps@ixmaps.ca:/home/ixmaps/backup (what is best practice for including a seeding sql file with repo?)
cd /home/ixmaps/backup
gunzip ixmaps_seed.sql.gz
psql ixmaps < ixmaps_seed.sql
```

#### Backups
```
mkdir /home/ixmaps/backup
mkdir /home/ixmaps/backup_daily
```

#### Crontab setup
User ixmaps
```
# download new Maxmind data file (at 2:00 on the 15th of each month)
0 2 15 * * /home/ixmaps/bin/download_maxmind.sh

# geocorrection of any new IPs (every 10 minutes)
*/10 * * * * /home/ixmaps/bin/corr-latlong.sh -n
# update cities to match lat and long (every 20 minutes)
*/20 * * * * php /srv/www/php-backend/application/controller/geo_update_cities.php > /home/ixmaps/log/geo_update_cities.log
# recheck all geocorrection of IPs (every day at 1:00)
0 1 * * * /home/ixmaps/bin/corr-latlong.sh -u
# update the traceroute_traits and annotated_traceroutes tables (every day at 3:00)
0 3 * * * /srv/www/php-backend/application/controller/derived_tables.php >> /home/ixmaps/log/derived_tables.log

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

# update database stats for website
*/5 * * * * /home/ixmaps/bin/db-stats.sh
```
User root
```
# autorenewl of SSL certs (every Monday at 1:00 and 1:05)
0 1 * * 1 /opt/letsencrypt/certbot-auto renew >> /var/log/le-renew.log
5 1 * * 1 service apache2 reload
```

#### Matomo/Piwik setup
We're running piwik on our dev server (dev.ixmaps.ca).
To creating a new site instance through at https://dev.ixmaps.ca/piwik/index.php, then add the relevant siteId variables are in /srv/www/website/\_includes/global-head.php

### IXmapsClient
https://github.com/ixmaps/IXmapsClient
```
sudo mkdir /srv/www/IXmapsClient
sudo chown ixmaps:ixmaps /srv/www/IXmapsClient
ln -s /srv/www/IXmapsClient /srv/www/website/
wget https://www.ixmaps.ca/IXmapsClient/IXmapsClient_v.1.0.6.macos.dmg
wget https://www.ixmaps.ca/IXmapsClient/IXmapsClient.1.0.6.win64.exe
wget https://www.ixmaps.ca/IXmapsClient/IXmapsClient.1.0.6.linux.tar.gz
```

#### TRsets setup
```
sudo mkdir /srv/www/trsets
sudo chown ixmaps:ixmaps /srv/www/trsets
ln -s /srv/www/trsets /srv/www/website/
git clone https://github.com/ixmaps/trsets.git /srv/www/trsets
```

### Misc
#### ip_addr p_status lifecycle
General pattern: N -> G -> F
```
1. GatherTr::insertNewIp sets it to 'N'
2. The cronjob corr-latlong script looks at 'N' (needs geolocation) and 'U' (unknown location). It then sets p_status to 'G' if corrected or 'U' ifnot corrected
3. The cronjob geo_update_cities.php calls IXmapsGeoCorrection::getIpAddrInfo to determine which IPs to update (those with p_status of 'G'). IXmapsGeoCorrection::updateGeoData then updates the city/country based on new lats and then sets the p_status to 'F'. So 'N' and G' only exist for a very short time.
```

## License
Copyright (C) 2020 IXmaps.
These scripts [github.com/ixmaps/ixmaps-bin](https://github.com/ixmaps/ecosystem) are licensed under a GNU AGPL v3.0 license. All files are free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, version 3 of the License.

These scripts are distributed in the hope that they will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details [gnu.org/licenses](https://gnu.org/licenses/agpl.html).
