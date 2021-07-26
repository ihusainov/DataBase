# Scripts

**FTP**

Copy to remote server over curl sftp
```bash
#!/bin/bash

/usr/bin/pg_dump confluencedb | gzip | curl -u "username:password" -k  "sftp://servername/pg_backup/confluence/`date '+%Y-%m-%d_%H-%M'`/db_confluencedb.gz" --ftp-create-dirs -T -
```


Copy to remote server over rsync and delete older folders and files
```bash
#!/bin/bash

mkdir /var/lib/postgresql/backup/`date '+%Y-%m-%d_%H-%M'`;
/usr/bin/pg_dump -Fc confluencedb > /var/lib/postgresql/backup/`date '+%Y-%m-%d_%H-%M'`/db_confluencedb.bak;

#----------------------------------------------------------

rsync -avqz --stats --delete --progress -e 'ssh -i /var/lib/postgresql/.ssh/postgres.ppk' /var/lib/postgresql/backup/ username@servername:/pg_backup/confluence/
find /var/lib/postgresql/backup/* -mtime +2 -exec rm {} \;


#-------------------End of the script-----------------------
```

