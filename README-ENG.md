# Sanic Micro Service

Sanic-based microservices infrastructure

## Introduce

One of the biggest problems faced by using python for web development is performance, which is a bit difficult to solve the C10K problem. Some asynchronous frameworks Tornado, Twisted, Gevent, etc. are designed to solve performance problems. These frameworks have improved performance a bit, but there are also all kinds of weird problems that are difficult to solve.

In python3.6, the official asynchronous coroutine library asyncio officially became the standard. While retaining convenience, performance has been greatly improved, and many asynchronous frameworks have used asyncio.

The earlier asynchronous framework is aiohttp, which provides the server side and the client side, and does a good encapsulation of asyncio. However, the development method is different from the most popular micro-framework flask. The development of flask is simple, lightweight and efficient. Combine the two and you have sanic.

The Sanic framework is an asynchronous coroutine framework similar to Flask, which is simple and lightweight, and has high performance. This project is a microservice framework based on sanic. Microservices are very popular recently. It solves the problem of complexity, improves development efficiency, and facilitates deployment.

It is the combination of these advantages, based on sanic, that integrates multiple popular libraries to build a microservice framework.

##Features

**Use the sanic asynchronous framework, simple, lightweight and efficient. **
* **Using uvloop as the core engine makes sanic as much as Golang in single-machine concurrency in many cases. **
* **Use asyncpg as the database driver to connect to the database and execute sql statement execution. **
* **Use aiohttp as Client to access other microservices. **
**Use peewee as ORM, but only for model design and migration. **
**Use opentracing as a distributed tracing system. **
**Use unittest for unit testing, and use mocks to avoid accessing other microservices. **
* **Use swagger as API standard, which can automatically generate API documentation. **

## Usage

[Example](https://github.com/songcser/sanic-ms/tree/master/examples)

#### Swagger API
![image](https://github.com/songcser/sanic-ms/raw/master/examples/images/1514528294957.jpg)

#### Zipkin Server
![image](https://github.com/songcser/sanic-ms/raw/master/examples/images/1514528423339.jpg)
![image](https://github.com/songcser/sanic-ms/raw/master/examples/images/1514528479787.jpg)

##Config

> Setting the configuration file is similar to Django, by setting the environment variable value SANIC_CONFIG_MODULE

````
export SANIC_CONFIG_MODULE='mysite.configs'
````

##Server

> Using the sanic asynchronous framework has high performance, but improper use will cause blocking, and the asynchronous library should be used for IO requests. **Add libraries with caution**.
> sanic uses uvloop asynchronous drive, uvloop is written in Cython based on libuv, and its performance is even higher than nodejs.

Function Description:

#### Before Server Start

* Create DB connection pool
* Create Client connection
* Create queue for log tracking
* Create opentracing.tracer for log tracking

#### Middleware

* Handle cross-domain requests
* Create span for log tracking
* Encapsulate the response in a unified format

#### Error Handler

Handle the thrown exception and return the unified format

#### Task

Create a span in the task consumption queue for log tracking

#### Asynchronous Handler
Since an asynchronous framework is used, some IO requests can be processed in parallel

Example:

````
async def async_request(datas):
    # async handler request
    results = await asyncio.gather(*[data[2] for data in datas])
    for index, obj in enumerate(results):
        data = datas[index]
        data[0][data[1]] = results[index]

@user_bp.get('/<id:int>')
@doc.summary("get user info")
@doc.description("get user info by id")
@doc.produces(Users)
async def get_users_list(request, id):
    async with request.app.db.acquire(request) as cur:
        record = await cur.fetch(
            """ SELECT * FROM users WHERE id = $1 """, id)
        datas = [
            [record, 'city_id', get_city_by_id(request, record['city_id'])]
            [record, 'role_id', get_role_by_id(request, record['role_id'])]
        ]
        await async_request(datas)
        return record
````
get_city_by_id, get_role_by_id are parallel processing.


#### Related Links
[sanic](https://github.com/channelcat/sanic)


## Model & Migration

> Peewee is a simple and small ORM. It has few (but expressive) concepts, making it easy to learn and intuitive to use.
>
> ORM uses peewee, just for model design and migration, and data manipulation uses asyncpg.

Example:

````
# models.py

class Users(Model):
    id = PrimaryKeyField()
    create_time = DateTimeField(verbose_name='create time',
                                default=datetime.datetime.utcnow)
    name = CharField(max_length=128, verbose_name="user's name")
    age = IntegerField(null=False, verbose_name="user's age")
    sex = CharField(max_length=32, verbose_name="user's sex")
    city_id = IntegerField(verbose_name='city for user', help_text=CityApi)
    role_id = IntegerField(verbose_name='role for user', help_text=RoleApi)

    class Meta:
        db_table = 'users'


# migrations.py

from sanicms.migrations import MigrationModel, info, db

class UserMigration(MigrationModel):
    _model = Users

    # @info(version="v1")
    # def migrate_v1(self):
    #     migrate(self.add_column('sex'))

def migrations():
    try:
        um = UserMigration()
        with db.transaction():
            um.auto_migrate()
            print("Success Migration")
    except Exception as e:
        raise e

if __name__ == '__main__':
    migrations()
```

* Run the command python migrations.py
* The migrate_v1 function adds the field sex, and the name field must be added first in the BaseModel
* The info decorator will create a table migrate_record to record migrate, the version must be unique in each model, use version to record whether it has been executed, and author, datetime can also be recorded
* The migrate function must start with **migrate_**

#### Related Links

[peewee](http://docs.peewee-orm.com/en/latest/)


##DB

> **asyncpg is the fastest driver among common Python, NodeJS and Go implementations**
>
> Use asyncpg as the database driver, encapsulate the database connection, and perform database operations.
>
> One reason for not using ORM for database operations is performance. ORM will have performance loss, and the asyncpg high-performance library cannot be used. The other is that a single microservice is very simple, the table structure is not very complicated, and simple SQL statements can be processed, and there is no need to introduce ORM.

Example:

````
sql = "SELECT * FROM users WHERE name=$1"
name = "test"
async with request.app.db.acquire(request) as cur:
    data = await cur.fetchrow(sql, name)

async with request.app.db.transaction(request) as cur:
    data = await cur.fetchrow(sql, name)
````

* The acquire() function is non-transactional, which can improve query efficiency for the use of non-transactions that only involves queries
* The tansaction() function is a transaction operation, and transaction operations must be used for additions, deletions, and changes
* The request parameter is passed in to get the span for log tracking
* **TODO** Database read and write separation

#### Related Links
[asyncpg](https://github.com/MagicStack/asyncpg)
[benchmarks](https://magic.io/blog/asyncpg-1m-rows-from-postgres-to-python/)

##Client

> Using the client in aiohttp, the client is simply encapsulated to access other microservices.

> Don’t create a session per request. Most likely you need a session per application which performs all requests altogether.
> A session contains a connection pool inside, connection reusage and keep-alives (both are on by default) may speed up total performance.

Example:

````
@app.listener('before_server_start')
async def before_srver_start(app, loop):
    app.client = Client(loop, url='http://host:port')

async def get_role_by_id(request, id):
    cli = request.app.client.cli(request)
    async with cli.get('/cities/{}'.format(id)) as res:
        return await res.json()

@app.listener('before_server_stop')
async def before_server_stop(app, loop):
    app.client.close()

````

For accessing different microservices, you can create multiple different clients, so that each client will keep-alives

#### Related Links

[aiohttp](http://aiohttp.readthedocs.io/en/stable/client.html)


## LOG & Distributed Tracing System

> Use official logging, the configuration file is logging.yml, and the sanic version should be 0.6.0 or above. JsonFormatter converts logs into json format for input to ES
>
> Enter OpenTracing: by offering consistent, expressive, vendor-neutral APIs for popular platforms, OpenTracing makes it easy for developers to add (or switch) tracing implementations with an O(1) configuration change. OpenTracing also offers a lingua franca for OSS instrumentation and platform-specific tracing helper libraries. Please refer to the Semantic Specification.

### Decorator logger

````
@logger(type='method', category='test', detail='detail', description="des", tracing=True, level=logging.INFO)
async def get_city_by_id(request, id):
    cli = request.app.client.cli(request)
````

* type: log type, such as method, route
* category: log category, the default is the name of the app
* detail: log details
* description: log description, defaults to the function's comment
* tracing: log tracing, the default is True
* level: log level, default is INFO

### Distributed tracing system

* OpenTracing is based on distributed tracing systems such as Dapper and Zipkin, and establishes a unified standard for distributed tracing.
* Opentracing tracks each request, records each microservice that the request passes through, and connects them in a chain, which is crucial for analyzing the performance bottleneck of microservices.
* Use the opentracing framework, but convert to zipkin format on output. Because most distributed tracing systems use thrift for communication in consideration of performance issues, in the spirit of simplicity and Restful style, RPC communication is not used. Output in the form of logs, you can use fluentd, logstash and other logs to collect and then input to Zipkin. Zipkin supports HTTP input.
* The generated span is first put into the queue without blocking, and the span of the queue is consumed in the task. The upsampling frequency can be added later.
* For DB, Client has added tracing

#### Related Links

[opentracing](https://github.com/opentracing/opentracing-python)
[zipkin](https://github.com/openzipkin/zipkin)
[jaeger](https://uber.github.io/jaeger/)


##API

> The api documentation uses the swagger standard.

Example:

````
from sanicms import doc

@user_bp.post('/')
@doc.summary('create user')
@doc.description('create user info')
@doc.consumes(Users)
@doc.produces({'id': int})
async def create_user(request):
    data = request['data']
    async with request.app.db.transaction(request) as cur:
        record = await cur.fetchrow(
            """ INSERT INTO users(name, age, city_id, role_id)
                VALUES($1, $2, $3, $4, $5)
                RETURNING id
            """, data['name'], data['age'], data['city_id'], data['role_id']
        )
        return {'id': record['id']}
````

* summary: api summary
* description: detailed description
* consumes: request body data
*proreduces: the return data of the response
* tag: API tag
* The parameter passed in consumers and produces can be peewee's model, which will parse the model to generate API data, and the help_text parameter in the field field to represent the reference object
* http://host:ip/openapi/spec.json to get the generated json data


#### Related Links

[swagger](https://swagger.io/)

##Response

When returning, do not return the response, but return the original data directly. The returned data will be processed in Middleware and returned in a unified format. The specific format can be [View](https://gist.github.com/songcser/ae8af65f33f34f09f265879e107cb584 )

## Unittest

> Unit test using unittest

Example:

````
from sanicms.tests import APITestCase
from service.server import app

class TestCase(APITestCase):
    _app = app
    _blueprint = 'visit'

    def setUp(self):
        super(TestCase, self).setUp()
        self._mock.get('/cities/1',
                       payload={'id': 1, 'name': 'shanghai'})
        self._mock.get('/roles/1',
                       payload={'id': 1, 'name': 'shanghai'})

    def test_create_user(self):
        data = {
            'name': 'test',
            'age': 2,
            'city_id': 1,
            'role_id': 1,
        }
        res = self.client.create_user(data=data)
        body = ujson.loads(res.text)
        self.assertEqual(res.status, 200)
````

* where _blueprint is the blueprint name
* In the setUp function, use _mock to register the mock information, so that the real server will not be accessed, and the payload is the returned body information
* Use the client variable to call each function, data is the body information, params is the parameter information of the path, and other parameters are the parameters of the route

### coverage

````
coverage erase
coverage run --source . -m sanicms tests
coverage xml -o reports/coverage.xml
coverage2clover -i reports/coverage.xml -o reports/clover.xml
coverage html -d reports
````

* coverage2colver is to convert coverage.xml to clover.xml, the format required by bamboo is clover.

#### Related Links

[unittest](https://docs.python.org/3/library/unittest.html)
[coverage](https://coverage.readthedocs.io/en/coverage-4.4.1/)
##Exception

> Use app.error_handler = CustomHander() to handle the thrown exception

Example:

````
from sanicms.exception import ServerError

@visit_bp.delete('/users/<id:int>')
async def del_user(request, id):
    raise ServerError(error='internal error', code='10500', message="msg")
````

* code: error code, 0 if there is no exception, other values ​​are exceptions
* message: status code information
* error: custom error message
* status_code: http status code, use standard http status code


