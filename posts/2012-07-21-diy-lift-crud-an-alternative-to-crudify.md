---
title:  DIY Lift CRUD - an alternative to CRUDify
date: '2012-07-21'
description:
categories: [code]
tags: [scala, lift, CRUD, Database]
layout: post
---

This article describes a simple approach for creating your own CRUD (Create, Read, Update, Delete) database screens in 
[Lift](http://liftweb.net/).

### Why build your own CRUD screens? 

Lift has a useful trait, CRUDify, that automatically adds CRUD functionality to any class.  
CRUDify is useful for rapid prototyping and can even be deployed to production where basic functionality is sufficient. 

You can customise much of the default behaviour by overriding the CRUDify methods. 
However, there is a practical limit to how far you can go with the customisation.

When your requirements diverge too much from the default CRUDify functionality, 
you can easily find yourself trying to coerce CRUDify to perform in ways in which it was not designed to perform. 
This can quickly lead to bloated, convoluted code that is hard to maintain. 
At this point it is often easier to abandon CRUDify and write your CRUD screens from the ground up.

### What is covered?
This article describes a simple approach for implementing a customised CRUD application.  
It covers how to organise your application into folders, files, classes, and methods.  A naming convention is 
provided for these artifacts as well as a simple model illustrating allowable screen transitions.  

The approach is demonstrated with an example consisting of code for a simple CRUD application with a single database entity called Event. 
Event contains the single field 'eventName'. 

This article also suggests coding patterns that can be used for implementation challenges 
such managing state on the server between successive page requests and using CSS Selector Transforms 
to efficiently render data as HTML.

The example uses the Lift Mapper trait to read and write data from the database.  
However, this approach could also equally easily use Lift's Record library or any other ORM.

### Prerequisites
This article assumes that the reader is familiar with basic Lift concepts.

As a minimum, you need to be comfortable with the techniques covered in Chapters 1-4 & 7 of [Simply Lift](http://stable.simply.liftweb.net/)
or the first eight chapters of [Exploring Lift](http://exploring.liftweb.net), up to the section on Mapper.  

### Download the code
You can download the code discussed in this article from [www.github.com/dph01/lift-CRUDBasic](http://www.github.com/dph01/lift-CRUDBasic)

### Running Version
A running version of this code can be seen at [www.damianhelme.com/crudbasic](http://www.damianhelme.com/crudbasic)

### Overview

The user interacts with each data entity with separate HTML pages for each of the following operations: 

  * Creating a new instance
  * Listing all existing instances
  * Editing an existing instance
  * Viewing (i.e. read only) an existing instance 
  * Deleting an existing instance.


The diagram below shows the allowable page transitions for the Event example. 
The screen flow is entered through New Event or List Events. Navigation to the View, Edit, and Delete pages are through links on one of the other pages.

![Screen Flow]({{urls.media}}/CRUDBasicScreenFow.gif)

The code is oragnised into a model file, a snippet file, and a series of HTML templates as shown below:

![Folder Structure]({{urls.media}}/folders.gif)

### Model

For the sake of this example, Lift's Mapper class is used to manage database access. Each entity has its own Scala file in the model package.
For example, the code for the Event entity (in /src/main/scala/code/model/Event.scala) is: 

    class Event extends LongKeyedMapper[Event] 
      with IdPK {
        def getSingleton = Event 
        
        object eventName extends MappedString(this, 30) 
            with ValidateLength {
          override def validations = 
            valMinLen(3, "Event name must contain at least 3 characters.") _ ::
              super.validations
        }
    }
    
    object Event extends Event 
      with LongKeyedMetaMapper[Event] {}


A simple validation rule was added to help test handling of form submission failures, which will be discussed later.

### HTML

The HTML pages for manipulating entities are derived from five Lift templates.
For the Event entity they are:

 * createevent.html
 * listevent.html
 * editevent.html
 * viewevent.hmtl
 * deleteevent.hmtl

The templates reside in sub-directories of src/main/webapp. There is one directory for each entity.
 For instance, the templates for Event are in: src/main/webapp/event. 

Access to these pages are defined in the SiteMap in Boot.scala:

    Menu("Create Event") /  "event" / "createevent",
    Menu("List Events") /  "event" / "listevent",
    Menu("Edit Event") /  "event" / "editevent" >> Hidden ,
    Menu("View Event") /  "event" / "viewevent" >> Hidden,
    Menu("Delete Event") /  "event" / "deletevent" >> Hidden,
 
editevent, viewevent, deleteevent are hidden menu items because access to 
these pages is via links on other pages. 

Note that there is a redundant 'event' in the path. One could have just had /event/create and /event/edit, however, having 'event' repeated in the filename (i.e. createevent) allows you to easily 
distinguish between multiple open files in an IDE such as Eclipse.

The contents of each of the HTML template is given below adjacent to the corresponding snippet.

### Snippet

Each entity has an 'Ops' (short for CRUD Operations) snippet class.

The Ops snippet class for each entity lives in its own file within the snippet package. 

For the Event entity this is src/main/scala/code/snippet/EventOps.scala. 

This class contains a single render method for each HTML template. The general structure of the class is as follows:

    class EventOps {
      def create = { ... }
      def edit = { ... }
      def list = { ... }
      def view = { ... }
      def delete = { ... }
    }


### The form processing lifecyle
When a user types a URL of a form into a browser (e.g. http://localhost:8080/event/createevent), the browser creates and sends a HTTP GET request to the server for the specified resource. 

Lift returns a HTML form to the browser with the 'action' attribute set to the URL the user has just requested.

    <form action="/event/createevent" method="post">
     ...
    </form>

When the form is submitted,
the browser creates and sends a HTTP PUT request to the same URL including the form data in the body of the request. 

Lift developers have to decide what to return to the user in response to the PUT.

Within the context of this approach, use the following convention: 

  * If the form validation fails (e.g. the name field contains less than three characters), return the same HTML form again with the 
input fields populated with the content that the user submitted. 
If the user clicks submit again the browser sends another PUT request and the
cycle continues.
  * If the form processing succeeds, redirect the browser to the
next page in the work-flow, which, in our case, is the List Events page.

When a form is processed, the snippet behind the form is therefore invoked at least twice; 
once for the initial GET request and once for every subsequent PUT request.

### Managing State

Each 'Ops' snippet class uses a RequestVar to pass the server-side 
state between successive HTML page requests. For the Event instance, this is declared as:

      object eventRV extends RequestVar[Event](Event.create)


RequestVar is used to pass Event instances between page requests in the following cases:
 
 * From the initial GET request of a create or edit form to the subsequent 
 PUT request that processes the submitted data (and also between successive PUT requests if they
 occur)
 * From the list page to a delete, view, or edit page
 * From a view page to an edit page
 
The RequestVar object is declared in a scope in which the related 'Ops' class method has access .
In this example the object is within the class scope, but it could also have been declared within file scope. 
In some more complex use cases file scope would be necessary to facilitate 
communication between multiple 'Ops' classes. 
Examples of such use cases will be provided in a future blog post. 

For a more detailed discussion of RequestVars, see my blog post:
[Understanding Lift's RequestVars](http://tech.damianhelme.com/understanding-lifts-requestvars).

### Create

Events are created through the createevent.html template:

    <div data-lift="lift:surround?with=default;at=content">
      <h2 class="alt">New Event</h2>
      <div data-lift="EventOps.create?form=post">
        <span id="hidden"></span>
        <table>
            <tr>
              <td>Event Name</td>
              <td><input id="eventname" type="text" /></td>
            </tr>
         </table>
        <input id="submit" type="submit" /> <br />
      </div>
    </div>

The corresponding snippet render method is:

    def create = {
      var event = eventRV.is                             
      "#hidden" #> SHtml.hidden(() => eventRV(event) ) & 
      "#eventname" #> SHtml.text(eventRV.is.eventName,   
                  name => eventRV.is.eventName(name) ) & 
      "#submit" #> SHtml.onSubmitUnit(processSubmit)     
    }
      
Most of this template is standard Lift form and snippet processing that's described in detail 
in Simply Lift and Exploring Lift.  

When the user sends the initial GET request on createevent.html and line (2) of the create snippet method is executed, 
the eventRV.is method is called for
the first time on this eventRV instance. 
Since eventRV has not been initialised, eventRV initialises itself with a new Event instance.
The Event instance is the result of calling Event.create which is the default function specified when eventRV is declared.

If create is called on a PUT request, 
which would occur on a form reload after validation has failed, 
eventRV.is returns the event that was previously set by the SHtml.hidden function at line (3). 

SHtml.hidden inserts a hidden input field into the HTML page and registers an associated 
server-side function <code>() => eventRV(event)</code> that Lift will call when the form is submitted.
This function sets the eventRV in the subsequent PUT request to contain the event instance 
that was used used in the current request. 

For more information on the pattern used here to pass an instance from one page request to another
see [Understanding Lift's RequestVars](http://tech.damianhelme.com/understanding-lifts-requestvars)

Note that the SHtml.hidden line comes before the other HTML callback functions (SHtml.text, SHtml.onSubmit). 
This ordering is important since Lift calls the functions in the same order they are 
shown here. 
EventRV needs to be set before setting the member variables of Event instance contained in eventRV.

In the 'create' method the SHtml.hidden line isn't strictly necessary, however, it's useful to include as a general rule.
If this line was omitted, all that would have happened here is that on a PUT request, eventRV would not have been set when it is first accessed.
A new Event instance would then be created. 

In the example, this would work since the content of the name field would be written to the 
new instance (via the closure on line 4) when the form is submitted.
The instance used on the previous request will be lost to garbage collection in the usual way.
The main advantage in this case of including the SHtml.hidden line 
would be that it prevents unnecessary garbage collection.

However, there are some use cases where the SHtml.hidden line is essential.
For instance, the Event class may have had some fields that were not set via the form. 
As an example, suppose that the Event class had a member variable containing a foreign key to a
Location instance (representing the many-to-one relationship in real-life 
where events are held at a particular location). 
If the 'createvent' page can only be invoked from 
within the context of a specific location (e.g. from a link embedded in the 'viewlocation' page), then when the event is created the location member variable should be set automatically.

The actual code to do this it bit too lengthy to show here but it would be possible
for the create event operation to know the location context from which it has been called and the 
EventOps.create function to set the Event location member accordingly. 

Without the location present in a form field and without the SHtml.hidden call in the EventOps.create,
the location setting would be lost if the form is submitted.

Finally, line (4) uses a common pattern for binding the fields of a Mapper entity to a HTML form.
Line (5) registers a function that Lift will call to process the contents of the form
when it is submitted:

    def processSubmit() = {
      eventVar.is.validate match {
        case  Nil => {
          eventVar.is.save 
          S.notice("Event Saved")
          S.seeOther("/event/listevent")
        } 
        case errors => S.error(errors)
      }
      // exiting the function here causes the form to reload
    }
  
These are standard Lift techniques, for more information see [Simply Lift](http://stable.simply.liftweb.net/#toc-Section-7.10).

If processSubmit succeeds, we're taken to the listevent.html page which in-turn invokes the EventOps.list method. 
Note that the SHtml.hidden closure called in EventOps.create will still have set eventRV, 
but since the list method does't use eventRV, it will be ignored.


### Moving between HTML page views 
When navigating between the list, view and edit pages, the event RequestVar is used to hold the Event instance in memory
 on the server between page requests. 
For example, in the list view, when an item's view link is clicked, 
the following view page will be 
rendered using the same event instance in memory as was used to render that line in the list 
view.  Similarly, from an event's view page, 
when the edit link on that page is clicked, the same event instance is used to render the subsequent edit page.

The following sections present the code that implements this mechanism.


### List

The listevent.html template is:

    <div data-lift="lift:surround?with=default;at=content">
      <h2>Events</h2>
      <a href="/event/createevent">New Event</a> <br />
      <div data-lift="EventOps.list">
        <table>
          <thead>
            <tr>
              <th>Event Name</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody id="eventlist">
            <tr class="row">
              <td class="eventName">Dummy Name</td>
              <td class="actions">Actions</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    

The list snippet is: 

    def list = {
      val allEvents = Event.findAll                             
      ".row *" #> allEvents.map( t => {                        
        ".eventName *" #> Text(t.eventName) &                  
        ".actions *" #> {                                      
            SHtml.link("/event/viewevent",                     
              () => eventRV(t), Text("view")) ++ Text(" ") ++  
            SHtml.link("/event/editevent",                     
              () => eventRV(t), Text("edit")) ++ Text(" ") ++  
            SHtml.link("/event/listevent",                     
              () => {t.delete_!}, Text("delete"))}             
      } )          
    }
      

This EventOps.list method uses a common pattern for displaying a list of entities based on a HTML template. 
For more information, see the example Binding To Children section of the Lift Wiki's 
[Binding Via CSS Selectors](http://www.assembla.com/spaces/liftweb/wiki/Binding_via_CSS_Selectors)).

All the events are loaded from the database into a list held in memory at line (2). 
Lines (5 - 11) render each event as a line in the table, with associated hyperlinks to the 'viewevent', 
'editevent' and 'deleteevent' pages.
The view and edit links each bind to the fuction '() => eventRV(t)'. 
This is called on the server when the link is clicked, setting the eventRV for the scope of the resulting 
page request to contain
the event instance corresponding to the line that was clicked.

### Edit

The editevent.html is:

    <div data-lift="lift:surround?with=default;at=content">
      <h2 class="alt">Edit Event</h2>
      <div data-lift="EventOps.edit?form=post">
        <span id="hidden"></span>
        <table>
          <tr>
            <td>Event Name</td><td><input id="eventname" type="text"/></td>
          </tr>
        </table>
        <input id="submit" type="submit /> <br />
      </div>
    </div>

The EventOps.edit snippet is:

    def edit = {
      if ( ! eventRV.set_? )                              
            S.redirectTo("/event/listevent")
      
      val event = eventRV.is                              
      "#hidden" #> SHtml.hidden(() => eventRV(event) ) &   
      "#eventname" #> SHtml.text(eventRV.is.eventName,     
                   name => eventRV.is.eventName(name) ) &  
      "#submit" #> SHtml.onSubmitUnit(processSubmit)       
    }


This snippet is very similar to the 'create' snippet discussed previously with the 
following differences.

At line (2), the eventRV is expected to be set. 
The user should have navigated here by clicking
on the edit link on either the list or view page. 
Both of those links have an associated `() => eventRV(event)` function that sets eventRV 
for this request. 
The case in which the user has typed in the edit page url directly is trapped. 

The edit snippet has the same 'SHtml.hidden' mechanism, however, in 
this case it performs an essential function.
The event instance has an 'id' member variable corresponding to the primary key of the record in the database 
(Event inherits this field from IdPK). The id field is assigned a value when save is 
first called on the instance. When the code edits and saves an existing event instance, 
Mapper uses the id field to know to update an existing record in the database rather than create a new one.

The id field is not rendered as a form field. 
When the form is submitted, if the SHtml.hidden field didn't exist, the event instance on which the other field 
closures are operating will have been newly created for the POST operation and as a result the id will not have been set.
Thus if the save were to be called on this event instance, a duplicate record will be written to the database.

### View
The viewevent.html template is:

    <div data-lift="lift:surround?with=default;at=content">
      <h2 class="alt">View Event</h2>
      <div data-lift="EventOps.view">
         <table>
            <tr><td>Event Name</td><td id="eventname">Dummy Name</td></tr>
         </table>
         <a id="edit" href=#>Edit</a>
      </div>
    </div>

The snippet code for the view operation is:

    def view = {
      if ( eventRV.set_? ) 
        S.redirectTo("/event/listevent")
         
      var event = eventRV.is
      "#eventname *" #> eventRV.is.eventName.asHtml &
      "#edit" #> SHtml.link("/event/editevent", () => eventRV(event), Text("edit"))
    }
      

This code is similar to the other CRUD operations presented so far.
The main difference is that MappedField's 'asHtml' method (eventName inherits from MappedField) is being used to render a read-only display of the name value.

### Delete
The deleteevent.html template is:

    <div data-lift="lift:surround?with=default;at=content">
      <h2 class="alt">Delete Event</h2>
      <div data-lift="EventOps.delete">
      <p>Are you sure you want to delete event: <span id="eventname"></span>?</p>
         <a id="yes" href=#>Yes</a>
         <a id="no" href=#>No</a>
      </div>
    </div>
    
The EventOps.delete snippet is:

    def delete = {
      if ( ! eventVar.set_? ) 
        S.redirectTo("/event/listevent")
        
      var e = eventVar.is
      "#eventname" #> eventVar.is.eventName &
      "#yes" #> SHtml.link("/event/listevent", () =>{ e.delete_!}, Text("Yes")) &
      "#no" #> SHtml.link("/event/listevent", () =>{ }, Text("No")) 
    }
    
The main technique to note here is that the deletion occurs with Lift calling the 
`() =>{ e.delete_!}` funtion registered with the 'Yes' link. 
This function forms a closure around the event instance that the user
wants to delete and calls the `Mapper.delete_!` function on that instance.

### Summary
While this approach may be more work than extending the CRUDify trait, it is straight forward, and provides full control over your CRUD forms. 

After using this approach a couple of times
I suspect you'll find it easy enough and quick enough for it to become your default
starting point for all future CRUD functionality.
 
Please leave comments, thoughts, questions etc. 

### Resources
  * [Simply Lift](http://stable.simply.liftweb.net/), David Pollak
  * [Exploring Lift](http://exploring.liftweb.net/master/index.html), Derek Chen-Becker, Marius Danciu and Tyler Weir
  * My blog post: [Understanding Lift's RequestVars](http://tech.damianhelme.com/understanding-lifts-requestvars)
