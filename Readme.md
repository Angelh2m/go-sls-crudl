# go-sls-crudl

This project riffs off of the [Dynamo DB Golang samples](https://github.com/awsdocs/aws-doc-sdk-examples/tree/master/go/example_code/dynamodb) and the [Serverless Framework Go example](https://serverless.com/blog/framework-example-golang-lambda-support/) to create an example of how to build a simple API Gateway -> Lambda -> DynamoDB set of methods.

## Code Organization
Note that, instead of using the `create_table.go` to set up the initial table like the AWS code example does, the resource building mechanism that Serverless provides is used.  Individual code is organized as follows:

* functions/post.do - POST method for creating a new item
* functions/get.do - GET method for reading a specific item
* functions/delete.do - DELETE method for deleting a specific item
* functions/put.do - PUT method for updating an existing item
* functions/list-by-year.do - GET method for listing all or a subset of items
* img/* - Images of DynamoDB tables to make this Readme easier to follow
* moviedao/moviedao.do - DAO wrapper around the DynamoDB calls
* data/XXX.json - Set of sample data files for POST and PUT actions
* Makefile - Used for dep package management and compiles of individual functions
* serverless.yml - Defines the initial table, function defs, and API Gateway events

Note that given the recency of Golang support on both AWS Lambda and the Serverless Framework, combined with my own Go noob-ness, I'm not entirely certain this is the best layout but it was functional.  My hope is that it helps spark a healthy debate over what a Go Serverless project should look like.

## Set Up
If you are a Serverless Framework rookie, [follow the installation instructions here](https://serverless.com/blog/anatomy-of-a-serverless-app/#setup).  If you are a grizzled vet, be sure that you have v1.26 or later as that's the version that introduces Golang support.  You'll also need to [install Go](https://golang.org/doc/install) and it's dependency manager, [dep](https://github.com/golang/dep).

When both of those tasks are done, cd into your `GOPATH` (more than likely ~/go/src/) and clone this project into that folder.  Then cd into the resulting `go-sls-crudl` folder and compile the source with `make`:

```bash
$ make
dep ensure
env GOOS=linux go build -ldflags="-s -w" -o bin/get functions/get.go
env GOOS=linux go build -ldflags="-s -w" -o bin/post functions/post.go
env GOOS=linux go build -ldflags="-s -w" -o bin/delete functions/delete.go
env GOOS=linux go build -ldflags="-s -w" -o bin/put functions/put.go
env GOOS=linux go build -ldflags="-s -w" -o bin/list-by-year functions/list-by-year.go
```
What is this makefile doing?  First, it runs the `dep ensure` command, which will scan your underlying .go files looking for dependencies to install, which it will grab off of Github as needed and place under a newly created `vendor` folder under `go-sls-crudl`.  Then, it'll compile the individual function files, placing the resulting binaries in the `bin` folder.  Keep in mind that this is an extremely dumb makefile in that it will compile every file every time as opposed to a smarter one that would only recompile files that have changed.  Blame my noobness with Golang.

If you look at the `serverless.yml` file, it makes references to those recently compiled function binaries, one for each function in our service and each one corresponding to a different verb/path in our API we're creating.  Deploy the entire service with the 'sls' command:

```bash
$ sls deploy
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Creating Stack...
Serverless: Checking Stack create progress...
.....
Serverless: Stack create finished...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service .zip file to S3 (15.19 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
...................................................................................................
Serverless: Stack update finished...
Service Information
service: go-sls-crudl
stage: dev
region: us-east-1
stack: go-sls-crudl-dev
api keys:
  None
endpoints:
  GET - https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/dev/movies/{year}
  GET - https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/dev/movies/{year}/{title}
  POST - https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/dev/movies
  DELETE - https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/dev/movies/{year}/{title}
  PUT - https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/dev/movies
functions:
  list: go-sls-crudl-dev-list
  get: go-sls-crudl-dev-get
  post: go-sls-crudl-dev-post
  delete: go-sls-crudl-dev-delete
  put: go-sls-crudl-dev-put
```

When done, you can find the new DynamoDB table in the AWS Console, which should initially look like this:

![Initial DynamoDB Table](/img/initialDynamoDBTable.jpg)

and your `<base URL>` will be of the format 'https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/dev/movies' where `XXXXXXXXXX` will be some random string generated by AWS.

The development cycle would then be:

* Make changes to the .go files
* Run `make` to compile the binaries
* Run `sls deploy` to construct the service and push changes to lambda
* Use `curl` to then interrogate the API as described below

## Using
Once deployed and substituting your `<base URL>` the following CURL commands can be used to interact with the resulting API, whose results can be confirmed in the DynamoDB console

### POST

```bash
curl -X POST https:<base URL> -d @data/post1.json
```
Which should result in the DynamoDB table looking like this:

![First Post DynamoDB Table](/img/firstPostDynamoDBTable.jpg)

Rinse/repeat for other data files to yeild:

![All Posts DynamoDB Table](/img/allPostsDynamoDBTable.jpg)

### GET Specific Item
Using the year and title (replacing spaces wiht '-' or '+'), you can now obtain an item as follows (prettified output):
```bash
curl https://<base URL>/2013/Hunger-Games-Catching-Fire
{
  "year": 2013,
  "title": "Hunger Games Catching Fire",
  "info": {
    "plot": "Katniss Everdeen and Peeta Mellark become targets of the Capitol after their victory in the 74th Hunger Games sparks a rebellion in the Districts of Panem.",
    "rating": 7.6
  }
}
```

### GET a List of Items
You can list items by year as follows (prettified output):
```bash
curl https://<base URL>/2013
[
  {
    "year": 2013,
    "title": "Hunger Games Catching Fire",
    "info": {
      "plot": "",
      "rating": 0
    }
  },
  {
    "year": 2013,
    "title": "Turn It Down Or Else",
    "info": {
      "plot": "",
      "rating": 0
    }
  }
]
```

### DELETE Specific Item
Using the same year and title specifiers, you can delete as follows:
```bash
curl -X DELETE https://<base URL>/2013/Hunger-Games-Catching-Fire
```
Which should result in the DynamoDB table looking like this:

![First Delete DynamoDB Table](/img/firstDeleteDynamoDBTable.jpg)

### UPDATE Specific Item
You can update as follows:
```bash
curl -X PUT https:<base URL> -d @data/put3.json
```
Which should result in the DynamoDB table looking like this:

![First Update DynamoDB Table](/img/firstUpdateDynamoDBTable.jpg)
