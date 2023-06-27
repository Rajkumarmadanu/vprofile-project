FROM mysql:8.0.33
LABEL "project"="Vprofile"
LABEL "Author"="Raj"

ENV MYSQL_ROOT_PASSWORD="Vprodpass"
ENV MYSQL_DATABASE="accounts"

ADD db_backup.sql docker-entrypoint-initdb.d/db_backup.sql
