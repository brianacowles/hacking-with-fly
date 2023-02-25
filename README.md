# Create a free Database with fly.io

Jumpstart your projects with a database! Fly.io is a cloud-based platform that offers scalable servers and database clusters that make app deployment simple. The feature-rich free tier and pay-as-you-go options make Fly.io the perfect platform to get experience with databases, app deployment, and backend development. 

## Creating the Database

### Prerequisites

First, you'll need a fly.io account. This can easily be done through the website: https://fly.io/

Since we're creating a database, you'll need to enter in your payment information. Don't worry; you'll never be charged without warning. If you'd like to read fly's motivation for requiring a credit card number, you can check it out here: https://fly.io/blog/free-postgres/.

Next, you'll need to install fly's command-line utility: `flyctl`. The installation method depends on your operating system, and the instructions are specified here: https://fly.io/docs/hands-on/install-flyctl/.

Once both of these steps are complete, you can sign into fly from the command-line:

```
$ flyctl auth login
```

### Making the Database

Start in a new empty directory:

```
$ mkdir restaurants-db && cd $_
```

You probably won't add any files to this directory during this tutorial, but it's important that you don't deploy any unwanted files accidentally. Additionally, if you take this project further, you'll want a directory to hold extra information you might need.

To create the database, we'll tell fly that we'd like to provision a new PostgreSQL database instance. We can do this with the following command:

```
$ fly pg create --image-ref flyio/postgres:14 
```

This will produce a series of prompts that help fly make our new database:

```
? Choose an app name (leave blank to generate one): restaurants-db
automatically selected personal organization: Briana Cowles
? Select region: Toronto, Canada (yyz)
? Select configuration: Development - Single node, 1x shared CPU, 256MB RAM, 1GB disk
```

First, fly will ask us to name our database. I chose "restaurants-db". Fly will automatically select the app's organization (typically just your personal account). For the region, I chose Toronto because it's close to Rochester, and we want to deploy our app close to our users. Finally, I chose the smallest possible size for our configuration, since we're just using Fly's free resources.

Fly will take a few seconds to build our database cluster, then it will provide us with the credentials. **Save these in a secure place; they won't be accessible later**.

### Making the Database Accessible Externally

When we created our database, you may have noticed that the hostname included the word "internal". By default, only apps within the same organization can access this database. This is a common practice because it makes the database more secure, but it can also make the database harder to access. In order to use our database on apps outside of fly (or even access it from a local machine), we need to make it available externally.

Currently, our database is like a house with no street number. If we want to be able to drive to it, we'll need an address. To give our database a public IP address, we'll use the following commands:

```
$ fly ips allocate-v4 --app restaurants-db
$ fly ips allocate-v6 --app restaurants-db
```

These commands will immediately print the new IP addresses, but we can double check that this worked by running `fly ips list --app restaurants-db`.

To save these changes, we just need to deploy the application. In order to do that, we'll need to know what major version of postgres we're using. This can be found with the following command:

```
$ fly image show --app restaurants-db

Image Details
MACHINE ID      REGISTRY                REPOSITORY      TAG     VERSION DIGEST                                                                  
9080e6eea61287  registry-1.docker.io    flyio/postgres  14      v0.0.34 sha256:9cfb3fafcc1b9bc2df7c901d2ae4a81e83ba224bfe79b11e4dc11bb1838db46e
```

Refer to the `TAG` to find the major version. In this case, it's 14. Getting the right version is important because there can be changes in the internal storage format between major versions of Postgres.

To deploy, run the following:

```
$ fly deploy . --app restaurants-db --image flyio/postgres:14
```

> There are two important details here. First, the `.` indicates that Fly is using the current directory to deploy the app. This is why it's important to start in an empty directory. Second, we're targeting the image flyio/postgres:14 because our database uses version 14 of postgres. If we used version 15 or 13, we'd have to replace that number.

This will take some time. Upon completion, Fly will generate the new hostname for our database: restaurants-db.fly.dev (this is the same as the previous hostname, but .internal has been replaced with .fly.dev)

We can use this new hostname to connect to our database from anywhere. The easiest way to connect is using Fly. This will open a new postgreSQL client on your machine. You may need to install `psql`, so check out [this article](https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/) for details:

```
$ fly postgres connect -a restaurants-db
```

If your connection fails, wait a few minutes. Fly might take some time to provision the resources for the database.

This isn't a Postgres tutorial, but I'll show two queries that are relevant to the next steps. We want to create a table to store our data. I'll be using restaurants in Rochester (I do actually recommend these if you're looking for new places to eat):

```PostgreSQL
CREATE TABLE restaurants(name varchar(50), cuisine varchar(30), favorite_item varchar(100));

INSERT INTO restaurants(name, cuisine, favorite_item) VALUES ('Sodam', 'Korean', 'Chicken Bulgogi'), ('El Latino', 'Dominican', 'Mofongo w/ Chicken'), ('Youkoso Sushi', 'Japanese', 'Nigiri Dinner'), ('OG Dumpling House', 'Chinese', 'Peanut Noodles');
```

The first query creates the `restaurants` table. The second inserts our data. We can view our data with `SELECT * FROM restaurants;`.

> This cheatsheet is useful: https://www.postgresqltutorial.com/postgresql-cheat-sheet/. The "Querying data from tables" section is a good place to start. This also discusses the psql command line interface (the CLI the fly command uses). Remember that all queries need to end with a semicolon. 
>
> If you're looking for a more user friendly way to connect to the database, check out [DBeaver](https://dbeaver.io/).


## Creating a Server

### Prerequisites

In order to complete this section, you’ll need access to Node.js and the Node Package Manager (npm). Follow these installation steps:
1.	Go to the official Node.js website at https://nodejs.org/en/download/.

2.	Select the appropriate installer for your operating system. If you're on Windows, download the .msi installer, and if you're on macOS or Linux, download the appropriate package manager installer.

3.	Once the installer has finished downloading, run it and follow the prompts to install Node.js. The default installation options should be sufficient for most users.

4.	After the installation is complete, open a terminal window and type the following command to check if Node.js and npm are installed: `node -v`, `npm -v`


### Making a Node App

First we’ll make a node application. Node.js is a software environment that allows you to run Javascript code outside of a web browser. This means you can run it right in your terminal. We’ll make use of the Express library, which will allow us to write a server that connects to our database.

Create a new directory and enter it. Then create a new file called `server.js`

```
$ mkdir restaurant-saver && cd $_
$ vim server.js
```

`server.js` will hold all the code for our server. We’ll write some basic code for now then expand upon it once we know it’s working. 

```JavaScript
// server.js
const express = require("express");
const app = express();
const port = 8080

app.get("/", (req, res) => {
   res.send("It's Alive!!!"); 
});

app.listen(port, () => console.log(`Listening on port ${port}`));
```

> Let’s talk about exactly what’s happening here. 
>
> First, we import the “express” module, a Node.js framework used to build web applications. Next we create a new instance of the express application. This instance is referred to as “app” and represents the HTTP server. Then we set the port number for the app to listen on. We will listen on port 8080, meaning we can access our app through http://localhost:8080.
>
> The next part of our server defines the real functionality. This is where we set up the routes of the app. Our route is defined at the root url (“/”) of the application. When a client makes a request to the root url, our server will respond with “It’s Alive!!!”.
>
> Finally, we start the express application by telling it to listen for incoming HTTP requests on our specified port. When the application starts successfully, it will print a message in our terminal indicating what port it’s listening on.

Before we can run our server, we need to complete our basic node setup. First, run `npm init -y`. This will generate a file called package.json with default values. This file manages our project’s metadata. Importantly, it lists our project’s dependencies, packages like “express” that we need to run the application.

Now that we have a `package.json`, we can install our dependencies. Run `npm install express –save` to install express. After this step, running `node server.js` will start our application. Navigate to http://localhost:8080 to verify that the app is working. You can stop the app at any time by hitting Ctrl+C in your terminal.

### Access PostgreSQL from the Server

To make our server access our database, we’ll need to install a new package and modify our code. First, install the `pg` package: `npm install pg –save`. This is a postgresql client made for Node.js. 

> You can always read more details about packages on the npm website: https://www.npmjs.com/package/pg. 

Now we need to make changes to `server.js`. First, import the `Pool` class from pg and create a new `Pool` with our database details. Use the details fly provided during database creation but use the new external hostname (e.g. restaurants-db.fly.dev).

```JavaScript
//server.js
const express = require("express");

// import the Pool class from pg and create a new Pool
const { Pool } = require('pg');

const pool = new Pool({
  host: 'my-postgres-db.fly.dev',
  user: 'postgres',
  password: '1a3ghaXpFb2kG8a',
  database: 'postgres',
  port: 5432
});
```

> Realistically, you would keep these as environment variables, values set in the operating system. Then you would access them like `process.env.DATABASE_URL`. You could also use the [env](https://www.npmjs.com/package/env) package. In general, DON'T keep these as plaintext in an application you care about.

Next, we need to change our route code. Instead of just saying "It's Alive!!!", we want the app to return the data in the `restaurants` table.

To make a request to the database, we'll use the pool object we created earlier. Let's get all the values in the `restaurants` table. In PostgreSQL, we write `SELECT * FROM restaurants`. We can make that query in the route for the root url of our server:

```JavaScript
//server.js

app.get('/', async (req, res) => {
    const result = await pool.query('SELECT * FROM restaurants');
    res.send(result.rows);
});
```

Notice the `await` keyword. This indicates that querying the database is an asynchronous function, meaning it can run in the background while the calling function continues executing. Adding the `await` keyword causes the code to stop executing until the asynchronous function is finished. Without the `await` keyword, our code would keep executing. Instead of sending our data to the client, the server would send a "Promise", a value that's not available yet but will be resolved eventually. Asynchronous functions execute differently from synchronous functions, so any time we use the `await` keyword, we need to mark the function it was used in as `async`. Notice that this keyword is used on the first line of the snippet.

There's only one more problem: the database query could fail. We need to wrap our query in a `try...catch` block to ensure that any problems we encounter don't crash our server.

```JavaScript
//server.js

app.get('/', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM restaurants');
    res.send(result.rows);
  } catch (error) {
    console.error(error);
    res.status(500).send('Internal server error');
  }
});
```

Inside the `catch` block, we print the error to our terminal and tell the user that there was a problem, indicated by an internal server error.

> We also set the status code of the response to 500. Status codes are standardized numbers that make it easier for developers to interpret the responses from a web application (e.g. 404 Not Found). Read more about HTTP status codes here: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status.

Put it all together and our new file looks like this:

```JavaScript
//server.js

const express = require("express");
const { Pool } = require('pg');

const pool = new Pool({
    host: 'restaurants-db.fly.dev',
    user: 'postgres',
    password: '7p34hWx3hCgd9k8',
    database: 'postgres',
    port: 5432
});

const app = express();
const port = 8080

app.get("/", async (req, res) => {
    try {
        const result = await pool.query('SELECT * FROM restaurants');
        res.send(result.rows);
    } catch (error) {
        console.error(error);
        res.status(500).send('Internal server error');
    }
});

app.listen(port, () => console.log(`Listening on port ${port}`));
```

You can make as many routes as you like that follow this format. Say you wanted to make a route that just gets a user's favorite items. You would change the route from "/" to "/favorites" (or something similar) then change the query to just get the `favorite_item` column: `SELECT favorite_item FROM restaurants`. 

## Deploying our app

The last step is deploying our app to make it public. Fly makes this process easy.

Fly will scan our source code, determine how to build a deployment image and identify any additional configurations needed. In the directory with `server.js`, run the following:

```
$ flyctl launch
```

This will generate the same series of prompts we saw earlier when we created our database. We already have a database, so don't add a Postgres or Redis db. The final question will ask us if we would like to deploy now - type `y` to make your app public. Deployment will take some time, but you can monitor it by navigating to your project on the fly dashboard and clicking on the Monitoring tab.

Once deployment is complete, head to the url hosted by fly. In this case it'll be https://restaurant-saver.fly.dev/.

> If your app isn't loading once it's deployed, make sure the port number matches the `internal_port` specified in `fly.toml`. We deliberately used 8080 here because it is the default for fly.

Any time you make changes to the app, you can deploy those changes with the following command:

```
$ fly deploy .
```

## Going Further

- Use environment variables to store your data, and configure those variables on the cloud
- Make more tables and query them from your app. Make views to simplify querying, add indexes to your tables, and experiment with roles on your database.
- Try adding parameters to your route. Ex. http://localhost:8080/?cuisine=Japanese
- Learn about pagination and implement it on your server.
- Keep the app entirely within fly. We made our database publically accessible (for the purposes of this tutorial), but we ended up just deploying our app with fly. Try to make the app without making the database external.

## Common Errors

`Error failed to fetch an image or build from source: Could not find image "docker.io/flyio/postgres:15"`

> the Fly app currently supports up to Postgres 14, instead use `fly deploy . --app <app-name> --image flyio/postgres:14`

`Error error machine <#> has a volume mounted and app config does not specify a volume; remove the volume from the machine or add a [mounts] configuration to fly.toml`

> say N when asked to migrate the machine into Fly Apps Platform

## Sources

https://fly.io/docs/postgres/getting-started/create-pg-cluster/

https://fly.io/docs/postgres/connecting/connecting-external/

https://fly.io/docs/languages-and-frameworks/node/
