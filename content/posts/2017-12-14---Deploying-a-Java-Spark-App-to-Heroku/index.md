---
title: Deploying a Spark Java App with a PostgreSQL Database to Heroku
date: "2017-12-14T00:00:00.00Z"
template: "post"
draft: false
slug: "/posts/deploying-a-java-spark-app-to-heroku"
category: "Web Development"
tags:
  - "Web Development"
description: "Spark is a simple and lightweight Java web development framework. This article shows how to deploy a Spark app with a Database on Heroku."
---

[Spark](http://sparkjava.com/) is a simple and lightweight Java micro-framework that allows you to quickly create web applications and APIs (not to be confused with [Apache Spark](https://spark.apache.org/)). There are plenty of resources on getting up and running with Spark such as their [Github repo](https://github.com/perwendel/spark) or the [official website](http://sparkjava.com/tutorials/).

Once you get comfortable with Spark, you might run into some difficulties deploying your application, especially if you are building your application using Gradle and it has a database. I had found some articles online on how to deploy a Spark app but I had trouble following them since they were either using Maven to deploy or didn’t address the issue of specifically deploying an application with a database. I’ve written this article to hopefully serve as a reference for those looking to deploy a Spark application to Heroku using Gradle.

# Setup
To follow along with this article, please clone [this](https://github.com/brianmarete/to-do-list-spark) repository on to you machine. Once you’ve cloned it, please follow the instructions in the README on setting up your database. This tutorial is written using Gradle version 4.8+

## Assumptions
I’m going to make the following assumptions:

- You are familiar with Java 8, PostgreSQL, Git, and Gradle.
- You have installed the above stated technologies on your machine.
- You have installed the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli) and have an Heroku account.

## Project Structure
The to-do list application follows the basic structure of a Gradle app:

```
 to-do-list-spark
    ├── build.gradle
    ├── .gitignore
    ├── Procfile
    └── src
        ├── main
        │    ├── java
        │    │    ├── App.java
        │    │    ├── Category.java
        │    │    ├── DB.java
        │    │    └── Task.java
        │    └── resources
        │         └── templates
        │             └── …
        └── test
            └── java
                 └── …
```
The contents of the `templates` and `test/java` folders are not really necessary for this tutorial so I didn't list them out. I also omitted listing the gradle folder and gradle wrapper files but they should be in the repository.

Note: While it is typically best practice to have Git ignore jar files, there is one jar file that Heroku needs to know about: `gradle-wrapper.jar`. Be sure to have the following line in your .gitignore :
```
!gradle-wrapper.jar
```

# Procfile and build.gradle
Let’s start by taking a look at the Procfile file:
```
web: ./build/install/todo/bin/todo
```
This is an important file when deploying to Heroku and it needs to be at the root of your project directory. According to Heroku:

> A Procfile contains a number of process type declarations, each on a new line. Each process type is a declaration of a command that is executed when a dyno of that process type is started.

So this single line tells heroku “When the web dyno starts, run the `todo` binary file in the `./build/install/todo/bin` folder". Dynos are virtual Unix containers where your application will run and are part of the Heroku architecture. You can read more about Procfiles, dynos, and general Heroku terminology [here](https://devcenter.heroku.com/articles/how-heroku-works).

But where does this binary file come from? To answer this, we need to take a look at the `build.gradle` file:

TODO: Add Gist

This is a very basic `build.gradle` file so feel free to modify it based on your application's needs. The key things to note are:

1. The stage task on line 25 is needed for any Spark application deployed to Heroku. We list the clean and installDist tasks as dependencies for this task. The installDist task is what creates the todo binary.
2. We set the applicationName variable to "todo" in line 23. The installDist task uses this variable to create the binary file. If, for example, you set the applicationName variable to "restaurant", a binary named restaurant would be created in the ./build/install/restaurant/bin/ directory. You need to create the Procfile with this in mind.

You can read more about deploying Gradle apps to Heroku [here](https://devcenter.heroku.com/articles/deploying-gradle-apps-on-heroku).

# App.java

The main method of our application is located in `App.java`. Here you'll find all the routes of our applcation. You may notice the following lines of code at the top of our main method:

TODO: Add Gist

When running locally, Spark uses port 4567 by default but when it is deployed on Heroku, we may be assigned a different port. The port assigned to our application by Heroku is stored in the `PORT` environment variable. Unless you also have an environmental variable called `PORT` on your local machine, our application assumes that if the `PORT` variable is null, we are running locally. Make sure to include the above code **before** any of your routes.

# Creating the Heroku Application

At this point we are ready to create an application on the Heroku platform, which we can conviniently do with the Heroku CLI. First login:

```
$ heroku login
```
If you would like to give the application a custom name on Heroku, substitute `NAME-OF-APP` in the following command with your custom name. If you omit it, Heroku will provide you with a randomly generated application name.

Make sure you are at the root of the project directory then run:

```
$ heroku create NAME-OF-APP
```
The previous command should have added a remote repository called “heroku” so commit any changes you may have made then push to heroku:
```
$ git push heroku master
```

This output will display a wall of text which is just information from Heroku while it is deploying our application.

Next, we need to add a Postgres database to the Heroku application:

```
$ heroku addons:create heroku-postgresql:hobby-dev
```

The second last line of the output of that command should be something along the lines of:
```
…
Created postgresql-objective-24718 as DATABASE_URL
…
```
Your output may vary, especially the `postgresql-objective-24718` part but it will be similar. This is the name of your database on Heroku. Take note of it because we will use it soon.

# DB.java

Working with the Java Database Connection API can be a bit challenging so we use the [Sql2o](http://www.sql2o.org/) framework to simplify our work. We need to define a static Sql2o instance which we use to open and close the connection to our database as well as running SQL queries. We initialize it in the `DB.java` file:

TODO: Add Gist

The bulk of the file is the initialization of the Sql2o instance in the static constructor. We use the Java URI class to help construct our JDBC connection string. Between lines 13 and 17, we’re creating a URI object which will be different depending on whether we’re running locally or on Heroku.

Heroku stores the credentials for our database in an environmental variable named `DATABASE_URL` which you can view on the terminal by running:

```
$ heroku config
```

The `DATABASE_URL` variable holds a string of the format `postgres://username:password@Database-Server:port/path-to-database`. If the `DATABASE_URL` variable is null (line 13) we know that we're running locally and we initialize the URI instance with the string `postgres://localhost:5432/to_do`. If you named your local database something other than "to_do", you should replace that in the string. If we're running on Heroku, we initialize the URI instance with the contents of `DATABASE_URL`.

Between lines 18 and 22, we’re creating variables that will be used for initialize the Sql2o instance. On line 21 and 22, we use the `getUserInfo()` method which returns the username and password in the format `username:password`. We split this string into an array and assign the first element to `username` and the second element to `password`. Also note that if we are running locally, `getUserInfo()` will return null because we don't have that information in the URI string. If your PostgreSQL database requires authentication, change line 21 and 22 to:

```java
String username = (dbUri.getUserInfo() == null) ? <YOUR-DATABASE-USERNAME-HERE> : dbUri.getUserInfo().split(“:”)[0];
String password = (dbUri.getUserInfo() == null) ? <YOUR-DATABASE-PASSWORD-HERE> : dbUri.getUserInfo().split(“:”)[1];
```
Make sure to replace `YOUR-DATABASE-USERNAME-HERE` and `YOUR-DATABASE-PASSWORD-HERE` with **your** database username and password respectively. For security purposes, I suggest storing your username and password as constants in another Java class that is ignored by your version control system.

## Pushing the Local Database to Heroku

Finally, we need to push our local database to Heroku. I hope you took note of the name of your Heroku database when we added the heroku-postgresql addon. If not, you can simply login to your Heroku psql terminal by running:
```
$ heroku psql
```
And it should display the name of your Heroku database on the first line of the output. You can then exit by typing `\q`. Once you have the name of your Heroku database, you are ready to push your local database to Heroku. My local database was called `to_do` and my Heroku database was called `postgresql-objective-2471` so I would run the following command on the terminal:

```
$ heroku pg:push to_do postgresql-objective-2471
```
You should replace `to_do` and `postgresql-objective-2471` with the names of your local and Heroku databases respectively. You'll see a lot of output so don't worry! If you see the following line:
```
 …
WARNING: errors ignored on restore: 1
```
Feel free to ignore this. It is related to our local machine’s permissions, which are different than on our remote Postgres database. Your app should work fine.

And we are done!
