mongoimport --uri "mongodb://127.0.0.1:27017" --db instagram --collection account --file /data/account/instagram.account11.json --jsonArray

mongoimport --uri "mongodb://127.0.0.1:27017" --db instagram --collection local_source --file /data/account/instagram.local_source_11.json --jsonArray


#### Create Collection anonymous (MobileProxy)
mongoimport --uri "mongodb://127.0.0.1:27017" --db instagram --collection anonymous --file /data/account/instagram.anonymous_11.json --jsonArray