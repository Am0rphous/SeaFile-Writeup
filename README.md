# SeaFile Writeup

I wanted to share some notes after strugling to setting up Seafile as a selfhosted Google Drive alternative. Selfhosting is nice if you value privacy of your files, and you want full control over your data. I landed on Seafile because it seemd to have a slick interface, had a free community version, and could be selfhosted in e.g. Docker.

In the end I didn't get to upload new files because of strange errors such as:
1. "Network error" when dragging files into the web site for upload
2. Error in the seafile container when connecting to the seafile-mysql container `Skip running setup-seafile-mysql.py because there is existing seafile-data folder. Waiting for mysql server to be ready: %s (2003, "Can't connect to MySQL server on 'db' ([Errno 111] Connection refused)")`
3. Database error in the seafile-mysql container: `6 [Warning] Aborted connection 6 to db: 'seahub_db' user: 'seafile' host: '172.18.0.4' (Got an error reading communication packets)`

