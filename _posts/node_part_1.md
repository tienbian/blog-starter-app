---
title: 'Build an API with Node.js + Typescript + TypeORM from scratch and auto deploy on AWS ECS by GithubAction (Part 1)'
excerpt: 'in a Node.js series'
coverImage: '/assets/blog/preview/cover.jpg'
date: '2021-01-08T05:35:07.322Z'
author:
  name: TienTran
  picture: '/assets/blog/authors/jj.jpeg'
ogImage:
  url: '/assets/blog/hello-world/cover.jpg'
---

```sh
const user = new User();
user.full_name = 'Tran Vu Quoc Tien';
user.signature = 'from ruby with love';
await user.save();
```

Since Node.js has introduced, it has steadily increased in popularity ever since. No backend developers can ignore it. As you are reading this post, I'm assuming that you are familiar with REST API, Postgresql, DB migrations, Docker.
Let's say that we want to create an API to support a fake web application based on Netflix. We will call it NextPlease.

# The setup 🛠
You need to have Node 10+ and npm 5.0+ installed.
Check your Node version by
```sh
node -v
```
Check your npm installed by
```sh
npm -v
```
No more waiting. Let's start now.
# Project initialization
First of all, we need to create a new folder named *next_please*.
From the terminal, getting into our new folder, assuming that we have _npm_ installed locally, we type
```sh
mkdir next_please
cd next_please
npm init -y
```
That last command will create a *package.json* file into our *next_please* folder. *package.json* is a file used to store project details and external modules setting (that will be installed inside a folder named *node_modules*).

Then we need to install and initialize some modules to support Typescript.
we use *--save-dev* because we only need typescript modules for *devDependencies* means local development.
```sh
npm install -g typescript
npm install --save-dev tsc ts-node typescript
tsc --init
```
As a result, now you have a *tsconfig.json* file, which is used to specify the compiler options (as you may know, in the end, we will compile the Typescript code into Javascript).

I recommend use *eslint* module to check the syntax for Typescript code, to improve readability, maintainability, and functionality errors. But we will not install it in this post because it has a bunch of settings that depends on your coding style.

For simplicity just override your *tsconfig.json* with the below snippet
```sh
## ~/next_please/tsconfig.json
{
  "compilerOptions": {
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "resolveJsonModule": true,
    "module": "commonjs",
    "esModuleInterop": true,
    "target": "es2017",
    "noImplicitAny": true,
    "moduleResolution": "node",
    "sourceMap": true,
    "outDir": "dist",
    "baseUrl": ".",
    "paths": {
      "*": [
        "node_modules/*"
      ]
    }
  },
  "include": [
    "src/**/*"
  ]
}
```
# A simple Express server
Now we will create a folder *src* (means source), where we store all our Typescript files that need to be compiled. 
Before start, let's install *express* and also typescript version
```sh
npm install express
npm install --save-dev @types/express
```
No need specify *--save* option on *npm install* since npm 5.0+

We start by coding our *app.ts* inside *src* folder. This file will take responsibility for our routing functionality.
We also make *index.ts* inside *src* folder, where we expose the express server to port 5000.
Ending to have something like this:
```sh
.
├── node_modules
├── src
│   ├── app.ts
│   ├── index.ts
├── package-lock.json
├── package.json
├── tsconfig.json
```
We will declare our first endpoint in *app.ts*
```sh
## ~/next_please/src/app.ts

import express from 'express';

const app = express();
app.use(express.urlencoded({extended: true}));
app.use(express.json());
app.get('/', (req, res) => res.send('Hello World!'));
export {app};
```
No need *body-parser* modules for reading the body of response since Express version 4.16+
Here is the first version of *app.ts*, which contains only a simple GET endpoint "/" , which, when called, responds with a *"Hello World!"*.
Next, go on with *index.ts*:
```sh
## ~/next_please/src/index.ts

import {app} from './app';
import {AddressInfo} from 'net';
const server = app.listen(5000, '0.0.0.0', () => {
  const {port, address} = server.address() as AddressInfo;
  console.log('Server listening on:', 'http://' + address + ':' + port);
});
```
Before testing our express server working or not, we will add some scripts to our *package.json* (only modify the main and scripts parts)
```sh
## ~/next_please/src/package.json

{
  "...",
  "main": "dist/index.js",
  "scripts": {
    "prebuild": "rm -rf dist/*",
    "build": "tsc && cp -rf package.json dist/package.json",
    "prestart": "npm run build",
    "start": "ts-node . --unhandled-rejections=strict",
  },
  "...",
}
```
Now we can start our server by entering 
```sh
npm run start
```
Running that command will trigger scripts execution by order:
- prebuild
- build
- prestart
- start

which finally compiles our typescript code into *dist* folder.
We can check everything is OK by access *http://localhost:5000*
As output, we can see *Hello World!*

# Installing the database

In this article, we will use PostgreSQL as our DB and TypeORM as our ORM(Object-relational mapping).

We need to install Pg client and typeorm modules not only for devDependence so:
```sh
npm install typeorm
npm install pg
```

Now, we create our database component inside the *src/db* folder
```sh
## ~/next_please/src/db/db.ts

import {createConnection} from 'typeorm';
export const connect = async () => {
  const connection = await createConnection({
    'type': 'postgres',
    'database': process.env.TYPEORM_DATABASE,
    'host': process.env.TYPEORM_HOST,
    'port': 5432,
    'username': process.env.TYPEORM_USERNAME,
    'password': process.env.TYPEORM_PASSWORD,
    'synchronize': false,
    'logging': true,
    'entities': [
      __dirname + '/models/*{.ts,.js}',
    ],
    'migrations': [
      __dirname + '/migrations/*.js',
    ],
    'cli': {
      entitiesDir: __dirname + '/models/',
      migrationsDir: __dirname + '/migrations/',
    },
  });
};

```
As you can see, we will use environment variables.
We need install *dotenv* module to make environment variables available in our app
```sh
npm install dotenv
```
We will create *.env* with the following variables for local development, we will need other values for production.
```sh
## ~/next_please/.env

TYPEORM_CONNECTION = postgres
TYPEORM_HOST = localhost
TYPEORM_USERNAME = your_pg_username
TYPEORM_PASSWORD = your_pg_password
TYPEORM_DATABASE = your_database_name
TYPEORM_ENTITIES_DIR = src/db/models/
TYPEORM_ENTITIES = src/db/models/**/*.ts
TYPEORM_MIGRATIONS = src/db/migrations/*.ts
TYPEORM_MIGRATIONS_DIR = src/db/migrations/
TYPEORM_MIGRATIONS_TABLE_NAME = custom_migration_table
```

Now we need to initialize the database connection in *app.ts*

```sh
## ~/next_please/src/app.ts

import {connect} from "./db/db";
import dotenv from 'dotenv';

dotenv.config();
connect();

const app = express();
...
export {app};
```
We will create 2 additional folders inside the *db* folder: *models* and *migrations*
models: contain the model classes
migrations: contain database migration files
For simplicity, we just make and focus on 1 model *Movie* in this article.

First of all, we make a migration file to create table *movies*
Just for sure, make *typeorm* command available global in local development
```sh
npm install -g typeorm
typeorm migration:create --name CreateMovieTable
```
We just override *timestamp-CreateMovieTable.ts* with below snippet
```sh
## ~/next_please/src/db/migrations/<timestamp>-CreateMovieTable.ts

import {MigrationInterface, QueryRunner, Table} from "typeorm";
export class CreateMoviesTable1584289576471 implements MigrationInterface {
    public async up(queryRunner: QueryRunner): Promise<any> {
        return await queryRunner.createTable(new Table({
            name: "movies",
            columns: [
                {
                    name: "id",
                    type: "integer",
                    isPrimary: true,
                    isGenerated: true,
                    generationStrategy: 'increment'
                },
                {
                    name: "title",
                    type: "varchar",
                    isNullable: false,
                },
                {
                    name: "plot_summary",
                    type: "text",
                    isNullable: false
                },
                {
                    name: "duration",
                    type: "integer",
                    isNullable: false
                }
            ]
        }), true);
    }
    public async down(queryRunner: QueryRunner): Promise<any> {
        return await queryRunner.dropTable("movies");
    }
}
```
To run the migration, we need to add some scripts to our *package.json*
```sh
## ~/next_please/package.json

"scripts": {
...
"migration:run": "ts-node ./node_modules/typeorm/cli.js migration:run",
"migration:revert": "ts-node ./node_modules/typeorm/cli.js migration:revert"
```
Now enter the command
```sh
npm run migration:run
```
If everything was OK, you should see the last messages and a record has been inserted into *custom_migration_table*
```sh
Migration CreateMoviesTable<timestamp> has been executed successfully.
```
Now we should have a table *movies* in our database. Next step, we will create a model Movie.
We create a new file *Movie.ts* inside */src/db/models/* folder
```sh
## ~/next_please/src/db/models/Movie.ts

import {BaseEntity, Column, Entity, PrimaryGeneratedColumn} from "typeorm";
@Entity('movies')
export class Movie extends BaseEntity{
  @PrimaryGeneratedColumn()
  id: number;
  @Column()
  title: string;
  @Column()
  plot_summary: string;
  @Column()
  duration: number;
}
```
Now we are ready to create REST endpoint for movies
# Create a Movies
We will make a POST /movies endpoint to create a movie.
We will use ActiveRecord patterns of TypeORM to save movie record. TypeORM also provides many features to support querying and associations.

```sh
## ~/next_please/src/app.ts

import express from 'express';
import {connect} from './db/db';
import dotenv from 'dotenv';
import {Movie} from './db/models/Movie';

dotenv.config();
connect();

const app = express();

app.use(express.urlencoded({extended: true}));
app.use(express.json());

app.get('/', (req, res) => res.send('Hello World!'));

app.post('/movies', async (req,res) => {
  const movie = new Movie();
  movie.title = req.body.title;
  movie.plot_summary = req.body.plot_summary;
  movie.duration = req.body.duration;
  await movie.save();
  res.send(movie);
});

export {app};
```
We can test our endpoint
```sh
curl --location --request POST 'localhost:5000/movies' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'title=test' \
--data-urlencode 'plot_summary=test' \
--data-urlencode 'duration=10000'
```
and we should have response
```sh
{"title":"test","plot_summary":"test","duration":"10000","id":1}
```
And that's it. We have just made an API that handles create a movie record.
Now we should prepare for deploying our API.


# Prepare Deployment
In the first step of deployment, let's make a Dockerfile.
We will build Dockerfile for production only
```sh
## ~/next_please/Dockerfile

FROM node:12.18.1
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:12.18.1
ENV NODE_ENV=production
WORKDIR /app
COPY package*.json tsconfig.json ./
COPY ecs.entrypoint.sh /usr/local/bin/ecs.entrypoint.sh
RUN chmod +x /usr/local/bin/ecs.entrypoint.sh
RUN npm install --only=production
COPY --from=0 /app/dist ./dist
EXPOSE 5000
ENTRYPOINT ["ecs.entrypoint.sh"]
CMD [ "npm", "run", "pm2" ]
```
Here we will use *pm2* to run our API on production. Let's install it
```sh
npm install pm2
```
and add a script into *package.json* to run the process when our container has run
```sh
## ~/next_please/package.json

"scripts": {
...
"pm2": "pm2 start dist/index.js --no-daemon"
```

We should run migration every time we deploy to make sure our DB changing to be executed on production DB.
Here is the *ecs.entrypoint.sh* file
```sh
## ~/next_please/ecs.entrypoint.sh

#!/bin/bash
set -e
# Run migrate
./node_modules/.bin/typeorm migration:run

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```
We also make a *.dockerignore* file
```sh
## ~/next_please/.dockerignore

node_modules
npm-debug.log
```

Then finish part 1.
In the next part, we will deploy our API to AWS ECS by GitHubAction.
# Source code
https://github.com/tienbian/next_please
