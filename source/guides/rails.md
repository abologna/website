## Getting Started with Ember.js and Ruby on Rails

Before getting started with this guide, you should have the latest version of Ruby on Rails installed
(which is 3.2.3 at the time of writing). If you don't already have Ruby on Rails, you can follow
the instructions available on the [Rails website](http://rubyonrails.org/download).

In this guide, we'll show you how to build a simple, personal photoblog application using Ember.js and Ruby on Rails.

### Concepts

Ember.js is a front-end javascript model-view-controller framework. It is designed to help you create ambitious web applications which run inside the browser. These applications can use AJAX as their primary mechanism for communicating with an API, much like a desktop or mobile application.

Ruby on Rails is a Ruby-based full stack web framework. It also uses a model-view-controller paradigm to architect applications built on top of it, but in a much different way than Ember.js. The differences are beyond the scope of this guide, but you can read about them in [Gregory Moeck's excellent "Sproutcore MVC vs Rails MVC"](http://gmoeck.github.com/2011/03/10/sproutcore-mvc-vs-rails-mvc.html). What is critical to understand is that Ruby on Rails runs on the server, not the client, and is an excellent platform to build websites and APIs.

In the next few steps, we'll create a Ruby on Rails application which does two distinct but equally important things: It acts as a host for the Ember.js application we will write, and it acts as an API with which the application will communicate. 

It's worth noting that it's not at all necessary to host an Ember.js application using Ruby on Rails. It can be served from any standard web server (or a local file.)


### Creating a New Project

Use the `rails` command to generate a new project:

```
rails new photoblog -m http://emberjs.com/template.rb
```

The -m option specifies a template on which to base your new project. We have provided one for Ember.js apps which does the following:

* Loads the `ember-rails` and `active_model_serializers` gems
* Runs `bundle install`
* Generates an appropriate `ember` directory structure inside `app/assets/javascripts/ember`
* Generates an `AssetsController` and supplies an appropriate route in order to serve your application
* Generates an appropirate `ApplicationSerializer` for your applications data models.
  
When rails has finished creating your application it will reside in the `photoblog` directory. Switch to this newly created directory:

```
cd photoblog
```

### Creating the Server-side Models

This part will be familiar to anyone who has done Ruby on Rails development before. We'll create two new models, Photo and Comment. We start by asking Rails to generate the scaffolding for a Photo object.

```
rails generate scaffold Photo title:string url:string
```

Rails will generate a database migration, model, controller, resource routes, and other helpful files for us to start using. It will actually create more than we need: By default, rails scaffolding will generate standard CRUD (Create/Retrieve/Update/Destroy) views for our new model. Since our Ember.js application is going to be providing these views, we can safely remove them.

```
rm -rf app/views/photos
```

We should also ask Rails to generate our comment object and remove its views as well.

```
rails generate scaffold Comment text:string
rm -rf app/views/comments
```

We should now describe the accessible fields and the relationship of our models to Rails. In app/models/photo.rb, add the appropriate lines below:


```ruby
class Photo < ActiveRecord::Base
  attr_accessible :title, :url
  has_many :comments
end
```

And in app/models/comment.rb:

```ruby
class Comment < ActiveRecord::Base
  attr_accessible :text, :photo_id
  belongs_to :photo
end
```

If we look inside `db/migrate`, you'll see the database migrations that have been generated for us. We'll need to modify the `<datetime>_create_comments.rb` file to reference our photo model. 

```ruby
class CreateComments < ActiveRecord::Migration
  def change
    create_table :comments do |t|
      t.string :text
      t.references :photo
      t.timestamps
    end
  end
end
```

We can now run `rake db:migrate` to run these migrations and set up our database.

```
→ rake db:migrate
==  CreatePhotos: migrating ===================================================
-- create_table(:photos)
   -> 0.0184s
==  CreatePhotos: migrated (0.0185s) ==========================================

==  CreateComments: migrating =================================================
-- create_table(:comments)
   -> 0.0015s
==  CreateComments: migrated (0.0016s) ========================================
```

Our server-side models are now setup and ready for use! What we've done here is basically create a  API server that allows basic CRUD actions for our photo and comment models.

### Creating our Client-side Models

Now that we have models set up to persist our data on the server, we need to describe them to Ember. ember-rails, included with our project template, provides generators to help us with this.

```
rails generate ember:model Photo title:string url:string
```

```
rails generate ember:model Comment text:string
```

This creates the appropriate Ember.js models in `app/assets/javascripts/ember/models`. We'll need to describe the relationship between them by hand. To do this, we can use `DS.hasMany` and `DS.belongsTo`. We pass string which represent the path of the model class, in this case, `Photoblog.Comment` and `Photoblog.Photo`, respectively.

```javascript
Photoblog.Photo = DS.Model.extend({
  title: DS.attr('string'),
  url: DS.attr('string'),
  comments: DS.hasMany('Photoblog.Comment')
});
```

```javascript
Photoblog.Comment = DS.Model.extend({
  text: DS.attr('string'),
  photo: DS.belongsTo('Photoblog.Photo')
});
```

That's it! ember-data now knows about the structure of our data.

### Configure the Data Store

Next, we should configure our Ember.js app's data store, which will handle saving and updating our data models for us. We do when the application is created in the user's browser. The code that does this was generated automatically by our template, and is stored in `app/assets/javascripts/ember/photoblog.js`. We'll modify it to look like this:

```javascript
//= require_self
//= require_tree ./models
//= require_tree ./controllers
//= require_tree ./views
//= require_tree ./helpers
//= require_tree ./templates

Photoblog = Ember.Application.create({
  store: DS.Store.create({
    revision: 4,
    adapter: DS.RESTAdapter.create({
      bulkCommit: false,
    })
  })
});
```

Here, we're telling our application that when created, it should set the store property to a new DS.Store configured with the latest revision and a new DS.RESTAdapter, with bulkCommit turned off. You can read more about these options in the [ember-data readme](https://github.com/emberjs/data).

### Creating the State Manager

Our Ember.js application will be managed by a state manager. The state manger handles what view is currently being displayed, as well as some other application login. It should be created at `app/assets/javascripts/ember/states/photo_states.js` and it will look like this:

```javascript
Photoblog.stateManager = Ember.StateManager.create({
  initialState: 'photos',

  states: {
    photos: Ember.State.create({
      initialState: 'index',

      index: Ember.State.create({
        view: Photoblog.IndexView.create()
	  })
    })
  }
  
});
```

Here, we're defining a state manager for our application. We set up our states object and include a single state, `photos`, which we also set set as the initial application state. The `photos` state itself has a single substate, called `index`. This substate is set as the initial substate for the `photos` parent state. The `index` substate has a single property, called 'view' which we are going to set to an index view. We haven't written that yet, so lets do that.

### Creating the Index View

To see all our photos, we need an write an index view which shows them. To do this, we'll create two new files, one at `app/assets/javascripts/ember/templates/photos/index.handlebars` and one at `app/assets/javascripts/ember/views/photos/index_view.js`. First, let's look at the the `app/assets/javascripts/ember/views/photos/index_view.js`, as it is much simpler.

```javascript
Photoblog.IndexView = Ember.View.extend({
  templateName: 'ember/templates/photos/index',
  controller: Photoblog.photosController
});
```

This is where we define the Ember.js object which manages the view. We simply provide it with a `templateName` property, which points to our handlebars template, and a `controller` property, which manages the view. Here's the template that defines what the index view looks like.

```
<h1>My Photoblog</h1>

{{#each controller}}
  {{#view contentBinding="this"}}
    <div class="photo">
      <h2>{{content.title}}</h2>
      <img {{bindAttr src="content.url"}}>
      <br>
	  
      {{#if content.comments.length}}
        <h3>Comments</h3>
        <ul>
          {{#each content.comments}}
            <li>{{text}}</li>
          {{/each}}
        </ul>
      {{/if}}
    </div>
  {{/view}}
{{/each}}
```

Let's go break this down and explain what's gong on.

```
<h1>My Photoblog</h1>

{{#each controller}}
```

Our view has a controller, the Photoblog.photosController, which will create in the next step. This is an array controller, so it implements the Ember.Enumerable interface. This means that we can loop over it's contents (each element of the array) using the `#each` Handlerbars experssion.

```
{{#view contentBinding="this"}}
```

For each photo managed by the photosController, we will create a subview with the following contents, and have its `content` property be bound to the photo object.

```
    <div class="photo">
      <h2>{{content.title}}</h2>
      <img {{bindAttr src="content.url"}}>
      <br>
```

Here, we reference our photo to get its title, and user bindAttr to set the `<img>` tag's `src` attribute to the photo's url.

```
      {{#if content.comments.length}}
        <h3>Comments</h3>
        <ul>
          {{#each content.comments}}
            <li>{{text}}</li>
          {{/each}}
        </ul>
      {{/if}}
```

Next, we see if there are any comments on the photo. If there are, we create a section and list for comments, and iterate through them. Note that in this `{{#each}}` expression, we aren't binding the comment object to the content property, so the context is automatically set to it. We create a new `<li>` for each comment with the comments text, and close out our `{{#each}}` iteration, list, and {{#if}}.

```
  {{/view}}
{{/each}}
```

Now that we're done, we close out the subview and our iteration block. Our view is complete.

### Creating our Client-side Controller

Controllers serve as a mediator between your views and models. We've already discussed that we're going to need an `Ember.ArrayController` to manage our photo objects, so let's create it. You can create a new controller using the `ember:controller` generator. We can also create a new array controller by invoking the generator with the `--array` option:

```
rails generate ember:controller photos --array
```

This will generate a new array controller called `Photoblog.photosController` inside the `app/assets/javascripts/ember/controllers/photos_controller.js` file. Note that this file also creates a class called `Photoblog.PhotosController`. This allows you to easily create new instances of the controller for unit testing without having to reset singletons to their original state.

The `Ember.ArrayController` provides us with all the functionality we need for now, so no extra code is needed.

### Loading the App

We've now gone through the process of describe out models, views, and controllers to Ember and Rails. Let's get things off the ground by viewing our application!

The Rails template that we based out application off of came with a Rails controller called AssetsController and an associated view and route. This is designed to simply serve our application content, which is basically an empty page with the javascript code which will launch and run our app.

The last thing for us to do is to add the bootstraping code for our app. In `assets/javascripts/application.js`, we should add the following:

```javascript
$(function() {
  var photos = Photoblog.store.find(Photoblog.Photo);
  Photoblog.photosController.set('content', photos);

  Photoblog.getPath('store.adapter').mappings = {
    comments: Photoblog.Comment
  };

  Photoblog.stateManager.goToState('photos.index');

  Photoblog.photosView = Ember.ContainerView.create({
    currentViewBinding: 'Photoblog.stateManager.currentState.view'
  });

  Photoblog.photosView.append();
});
```

We're doing a few things here. First, we're getting all the photos in our data store, and setting the content of our `photosController` to the results array. Next, we set the data store's adapter mappings so that it knows comments are `Photoblog.Comments`. We then move to our initial state, and create an `Ember.ContainerView` with a `currentView` property that is bound to our current state's `view` property. Finally, add the `photosView` to the page.

You can now view the app in your browser by running `rails server` going to `http://localhost:3000`. You should see something like this:

![First site screenshot](/images/rails_site_1.png)

There's our title, but there's no content! We need to add some photos first, of course.

### Adding Photos

We need to add the ability to add photos to our application in order to see some on the index page. First, let's verify everything is working as expected by sending a POST request to our API with a new photo object. Run the following command:

```
curl 
```

Now ensure the server is running and reload the page. You should see the photo with it's title listed on the comments page. You'll also see the logs in the console where your server is running, showing the request being handled.


### Troubleshooting

We'll update this page with common issues as they come up. In the mean time, see our [Ember.js community](/community) page for more info on how to get help.