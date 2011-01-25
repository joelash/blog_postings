---
type: regular
post-id: 2909076530
state: draft
format: markdown
tags: clojure, compojure, web, trinidad
date: 01/23/2011
title: deployment of a compojure web app
---

Recently one of my projects at work has me developing a web application in [clojure](http://clojure.org/) using [compojure](https://github.com/weavejester/compojure). I've been pairing on our application for a few weeks with a few different people at work, a lot of this time has been spent learning the various tools; emacs, clojure, compojure and leiningen. (I'm not going to talk about these much, but my colleague, [Gaz Jones'](http://blog.gaz-jones.com/) blog was very helpful.) 

After following the compojure [getting started](https://github.com/weavejester/compojure/wiki/Getting-Started) doc we had ourselves a functional application, complete with several static files (javascript, css and images). Our next step was setting up our application to be deployed to our web servers. At work we have our web servers setup using [trinidad](https://github.com/calavera/trinidad) (which uses tomcat under the covers) as a container for our jruby apps (mainly written in rails with a few in sinatra). Ideally, we would like to use trinidad to deploy our new clojure app as well. In this post I'll explain the various things we did with our application to get it deployable to trinidad. I've built a [simple example application](https://github.com/joelash/deployable-web-clj-example) that can run in development mode via jetty or can be run via trinidad.

## where to start ##

Checkout the [code](https://github.com/joelash/deployable-web-clj-example). Take a look and see if you understand, it's pretty self explanatory, but here are some of the things we found more difficult to determine. 

### what's different for deployment? ###

First, you need a servlet class which will be use by the running container to start the application as jetty is only used for development. There's no need to show you this because there's nothing special about it, just stick to what it does and don't forget you need to `(:gen-class)`, otherwise the servlet will not be picked up as a service that can be started via trinidad.

### what's with all the wrapping? ###

You'll notice that the `app` var wraps our `routes` var with quite a few different pieces of middleware. Let's take a look at what all we're doing. The single most important piece of middleware is `context/wrap-context`. This is one of the things that's necessary to run our compojure app as a servlet. What this middleware wrapper does is removes our application's name from the URI so that'll match our routes; since our routes don't account for this. You'll notice that we've written some middleware, the `wrap-if` method, which allows us to only use certain wrapper while when we're in development mode.

Note: If you look at the `context.clj` file, you'll notice there is also a `context/link` method. This will add the context path to the links in our project. In the example project you can see this in the views file `layouts.clj` where we're linking to our css file.

### make WAR (not love?) ###

Now we need to make a WAR that we can deploy from our code; to do this we decided to use [leiningen-war](https://github.com/alienscience/leiningen-war). We decided to use this over some of the other alternatives because it allowed us more configuration of the WAR, which we needed in order to get our static files served correctly. To add leiningen-war to our project we had to add this line to our `project.clj` (version 0.0.12 is the current version at the time of this writing):

		:dev-dependencies [ul.org.alienscience/leiningen-war "0.0.12"]

While we're in the `project.clj` file there were a few other changes that are needed to build our WAR. First, we need to tell leiningen-war what namespace it can find the service in, to do this we added the following line:

		:aot [clj-deployable-web.deploy.servlet]
		
Leiningen-war will use this when it creates the `web.xml` file for the WAR. The last configuration step for leiningen-war is to tell it where our static files are (web-content). To do this we added this line to `project.clj`:

		:war {:web-content "html"}
		
Sweet, now we're all set. We are going to have to do something special in our `web.xml` to actually serve the static files but we're going to first let leiningen-war create one and then we'll edit it. Leniningen-war will only create one if one does not already exist. Let's go ahead and have leiningen-war create this so we can edit it, run:

		$ lein web-xml
		
Now edit the `web.xml` and add this `servlet-mapping` before the one for our app:
		
		<servlet-mapping>
			<servlet-name>default</servlet-name>
			<url-pattern>/public/*</url-pattern
		</servlet-mapping>
		
Since we've put all our static content under the public directory we can easily add a new mapping that will not go to our application and use tomcat to server the files from that folder. 

### configure trinidad ###

Awesome, we're almost set. Now all we need to do is configure trinidad to run our app. This file does not belong as part of the application, but as part of your deployment process for trinidad and all its applications. I've added one in the repo just so it can be used. If you already have a configuration file for trinidad, you'd have to add the following lines under `web_apps:` to add our new app:

		clj-web:
			default_web_xml: web.xml
			context_path: '/clj-example.war'
			web_app_dir: 'clj-deployable-web-1.0.0-SNAPSHOT.war'

What this all means: the `default_web_xml:` is telling trinidad that your `web.xml` is located at `WEB-INF/web.xml` not at `WEB-INF/config/web.xml`, which is trinidad's default. The `context_path:` tells trinidad what you want the context path to be, this will show up after  `http://<host>:<port>`. It must end in '.war' as this is how trinidad determines your application is a WAR, but don't worry the '.war' won't show up in the URI. Lastly, `web_app_dir:` is the path to your WAR file.
	
### run it ###

You can move the `trinidad_config.yml` and `clj-deployable-web-1.0.0-SNAPSHOT.war` to a new directory if you'd like or just run it from here. But to run it do this:

		$ CLJ_WEB_ENV=production jruby -S trinidad --config trinidad_config.yml
		
Boom! It works. Notice the border, we have css!

### things learned / to note ###

* Using an `ENV` var for the application environment isn't the standard way to configure a WAR; this was done for simplicity in this example. Feel free to do this however you find best.

* When defining your routes it is important that you add `(compojure.route/not-found)` otherwise if any route (including a static file) is not found you'll receive a 500 error for the entire page.

* File names should have underscores for word breaks, but the namespace should have dashes.
