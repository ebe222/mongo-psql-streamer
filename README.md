
mongo-postgresql-streamer is used to make the bridge between PostgreSQL and MongoDB.
It acts as a river from MongoDB to PostgreSQL and is a direct

This connector is the successor of `PG Mongo Connector <https://github.com/Maltcommunity/mongo-connector-postgresql>`__
and is based on `mongo-postgresql-streamer <https://github.com/Maltcommunity/mongo-postgresql-streamer>`__

DISCLAIMER
----------

Please note: all tools and scripts in this repo are released for use "AS IS" without any warranties of any kind,
including, but not limited to their installation, use, or performance. We disclaim any and all warranties, either
express or implied, including but not limited to any warranty of noninfringement, merchantability, and/ or fitness for
a particular purpose. We do not warrant that the technology will meet your requirements, that the operation thereof
will be uninterrupted or error-free, or that any errors will be corrected.
Any use of these scripts and tools is at your own risk. There is no guarantee that they have been through thorough
testing in a comparable environment and we are not responsible for any damage or data loss incurred with their use.
You are responsible for reviewing and testing any scripts you run thoroughly before use in any non-testing environment.

# System Overview and prerequisites


`mongo-postgresql-streamer` creates a pipeline from a MongoDB cluster to PostgreSQL. It synchronizes data in MongoDB
to the target then tails the MongoDB oplog, keeping up with operations in MongoDB in real-time. It has been tested with
Java 8 and PostgreSQL 9.5.

## Installation


To install mongo-psql-streamer:
- clone the repository
- compile the application with
```
    mvn install
``` 
- the executable is located in the target folder. You can run it with the following command
```
    java -jar target/mongo-postgresql-streamer*.jar
```
That's all. But of course you'll need to override some configurations and we will explain it in the next sections.

## Using this connector

If you want to override configurations, you can pass arguments like this:
```
  java -jar target/mongo-postgresql-streamer*.jar --mongo.connector.identifier=members --mappings=mappings-members.json --server.port=8089
```

Alternatively, you can use a configuration file (in yml or properties format) like this 

```
  java -jar -Dspring.config.location=<path-to-file> target/mongo-postgresql-streamer*.jar 
```

## Options

|Option   | Default value   | description  |
|---|---|---|
|mongo.admin.database   | test   | The database where the connector store a checkpoint to remember the last oplog that was already processed|
|spring.datasource.url | jdbc:postgresql://localhost:5432 | the jdbc url to connect to postgresql |
|spring.datasource.username |  | username to login on pgsql |
|spring.datasource.password | | I think you already know what is this field |
|spring.datasource.platform | | this value should always be postgresql (mandatory) |
|mongo.connector.identifier | streamer | if you have different connectors, use different identifiers so that they don't have conflicts in the mongooplog collection  |
|mongo.connector.forcereimport | false | Use this argument if you want to force a new reimport of your schema and data  |
|mappings | mappings.json | The path to your mapping file |
|mongo.uri | mongodb://localhost:27017 | The connection url to your mongodb database |
 
 This option's list is not exhaustive.
 

## Mapping file

This connector use its own mapping file to determine the fields that should be written in PostgreSQL and their types.
This file should be named mappings.json. Here is a sample :

```
  {
  "DatabaseName":{

    "tablename":{
      "pk":"id",
      "_id": {
        "dest": "id",
        "type": "TEXT"
      },
      "idOfAnotherTableNotExisting": {
        "dest":"id_of_another_table_not_existing",
        "ref": {"active": false, "referrence": "another_table_not_existing", "referrence_pkey": "id", "referrence_alias": "fk_another_table_not_existing_id",
          "foreign_key": "fk_tablename_another_table", "key": "id_of_another_table_not_existing"},
        "type":"TEXT"
      },
      "stringColumn": {
        "dest":"string_column",
        "type":"TEXT"
      },
      "booleanColumn": {
        "dest":"boolean_column",
        "type":"BOOLEAN"
      },
      "integerColumn": {
        "dest":"integer_column",
        "type":"INTEGER"
      },
      "floatColumn": {
        "dest":"float_column",
        "type":"FLOAT"
      },
      "ArrayOfIds": {
        "dest":"TableForArray",
        "fk": "tablename_id",
        "type": "_ARRAY"
      },
      "timestampColumn": {
        "dest":"timestamp_column",
        "type":"TIMESTAMP"
      },
      "anArrayOfInteger": {
        "dest":"integer_array_table",
        "fk": "tablename_id",
        "type": "_ARRAY_OF_SCALARS",
        "valueField": "year"
      }
    },

    "table_for_array":{
      "pk":"id",
      "_id": {
        "dest": "_id",
        "type": "TEXT"
      },
      "tablename_id": {
        "dest":"tablename_id",
        "ref": {"active": true, "referrence": "tablename", "referrence_pkey": "id", "referrence_alias": "fk_tablename_id",
          "foreign_key": "fk_table_for_array_tablename", "key": "tablename_id"},
        "type":"TEXT"
      },
      "$oid": {
        "dest":"not_currently_existing_table_id",
        "ref": {"active": false, "referrence": "not_currently_existing_table", "referrence_pkey": "id", "referrence_alias": "fk_not_currently_existing_table_id",
          "foreign_key": "fk_table_for_array_not_currently_existing_table", "key": "not_currently_existing_table_id"},
        "type":"TEXT"
      }
    },

    "integer_array_table":{
      "pk":"id",
      "_id": {
        "dest": "_id",
        "type": "TEXT"
      },
      "tablename_id": {
        "dest":"tablename_id",
        "ref": {"active": true, "referrence": "tablename", "referrence_pkey": "id", "referrence_alias": "fk_tablename_id",
          "foreign_key": "fk_integer_array_table_tablename", "key": "tablename_id"},
        "type":"TEXT"
      },
      "year": {
        "dest":"year",
        "type":"INTEGER"
      }
    }
  }
}


```
Please notice the following :

- The ``pk`` field is mandatory and should point to the destination's primary key
- By default, the ``dest`` field is set to the original Mongo field name.
- If the original document in mongodb has a embedded document, everything is flattened to be inserted in PostgreSQL
- One can define indices in two different ways : Using the array ``indices`` and a SQL definition or autogenerate index
 by setting the ``index`` field to true
- You can reference other table just like in ``idOfAnotherTableNotExisting`` by using ``ref``, it's set to `false` 
due to the table not existing or if the other table doesn't have an id that is set on ``idOfAnotherTableNotExisting``,
 otherwise we set it to `true` just like on ``tablename_id`` in ``table_for_array``
The connector also supports arrays of documents. Let say your Mongo database stores the following documents :

```

    {
        "posts":{
            "name":"Check out the mongo -> postgres connector",
            "content":"Inspiring blog post",
            "comments":[
                {
                    "user":"Elon Musk",
                    "comment":"What a revolution !"
                },
                {
                    "user":"Kevin P. Ryan",
                    "comment":"Nice !"
                }
            ]
        }
    }
```
To allow the connector to map the post objects AND its comments, you should use the following mapping :

```

    {
        "my_mongo_database":{
            "posts":{
                "pk":"id",
                "_id":{
                    "dest":"id",
                    "type":"TEXT"
                },
                "content":{
                    "type":"TEXT"
                },
                "comments":{
                    "type":"_ARRAY",
                    "fk":"post_id"
                }
            },
            "comments":{
                "pk":"id",
                "post_id":{
                    "dest":"post_id",
                    "type":"TEXT"
                },
                "user":{
                    "dest":"user",
                    "type":"TEXT"
                },
                "comment":{
                    "dest":"comment",
                    "type":"TEXT"
                }
            }
        }
    }
```
Please notice the following :

- The type ``_ARRAY`` is used to indicate to the connector that the field is an array
- The additional field ``fk`` is provided to indicate to the connector where to store the root document id. This field is mandatory for an array
- The comments' mapping declares only the primary key but no mapping exists. The connector will generate the identifier automatically
- The foreign key must be declared in the comments table so it's created with the schema initialization

Finally, the connector supports arrays of scalar. Let say your Mongo database stores the following documents in the ``timeline`` collection :

```

    {
    	"author": "507f1f77bcf86cd799439011",
    	"posts": [{
    		"name": "Check out the mongo -> postgres connector",
    		"tags": [
    			"Awesome",
    			"Article",
    			"Postgres"
    		]
    	}]
    }
```
One can use the following mapping :

```

    {
    	"my_mongo_database": {
    		"timeline": {
    			"pk": "id",
    			"_id": {
    				"dest": "id",
    				"type": "TEXT"
    			},
    			"author": {
    				"type": "TEXT"
    			},
    			"posts": {
    				"type": "_ARRAY",
    				"dest": "timeline_posts",
    				"fk": "timeline_id"
    			}
    		},
    		"timeline_posts": {
    			"pk": "id",
    			"_id": {
    				"dest": "id",
    				"type": "TEXT"
    			},
    			"name": {
    				"type": "TEXT"
    			},
    			"tags": {
    				"dest": "timeline_posts_tags",
    				"type": "_ARRAY_OF_SCALARS",
    				"fk": "timeline_post_id",
    				"valueField": "tag"
    			}
    		},
    		"timeline_posts_tags": {
    			"pk": "id",
    			"_id": {
    				"dest": "id",
    				"type": "TEXT"
    			},
    			"tag": {
    				"type": "TEXT"
    			}
    		}
    	}
    }
```
Contribution / Limitations
--------------------------

most of the credits goes to the original tool creator, we only added changes that allow us to add references from other tables to connect it together.
the `test` folder have been removed due to the changes we made having conflict with it.