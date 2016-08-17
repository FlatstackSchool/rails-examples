# Base JS component

Base component, to provide development team the ability to write so javascript code in the same style. This will make it easier to understand someone else's code and reduce development time.

```ruby
# app/views/layouts/application.html.slim
= javascript_tag "window.App = {}"
```

```coffee
# app/assets/javascripts/application.coffee
...
#= require ./components/all
```

```coffee
# app/assets/javascripts/components/all.coffee
#= require ./component
...
```

```coffee
# app/assets/javascripts/components/component.coffee
class App.Component
  constructor: (el, props = {}) ->
    @refs ?= {}
    @$refs = {}
    @el = el
    @$el = $(el)
    @props = props
    @_initRefs()
    @initialize()
    @bindings()

  initialize: ->
    #template method

  bindings: ->
    #template method

  _initRefs: =>
    @$refs[name] = $(selector, @$el) for name, selector of @refs
```
#### variables:
* el - HTML DOM selector(root element).
* $el - jQuery element
* @refs - object that contains the data selectors which are needed for use in a component.
  * Key - name that will be used for access to the selector
  * Value - HTML DOM selector
* @$refs - object that contains the jQuery elements from @refs.
* props - options which can be transferred from the parent component

#### functions:
* initialize - use this function if you need to initialize other components within the component.
* bindings   - use this function if you need to listen to some events(change, click...)
* _initRefs  - at this function it will be initialize all the variables, which are specified in the `@refs` config. Please note that the search for variables is only within a component.

## Simple example

```coffee
# app/assets/javascripts/components/all.coffee
#= require ./component
#= require ./comments/form
#= require ./comments/list
#= require ./comments/box
...
```

```coffee
# app/assets/javascripts/components/comments/box.coffee
class App.CommentsBox extends App.Component
  refs:
    form: ".comments-form"
    list: ".comments-list"

  initialize: ->
    @_initComponentForm()
    @_initComponentList()

  _initComponentForm: =>
    @form = new App.CommentsForm @$refs.form, onSubmit: @prependComment

  _initComponentList: =>
    @list = new App.CommentsList @$refs.list

  prependComment: (comment) =>
    @list.addComment(comment)

$ ->
  new App.CommentsBox(".comments") if $(".comments").length
```

```coffee
# app/assets/javascripts/components/comments/form.coffee
class App.CommentsForm extends App.Component
  refs:
    textInput: ".comments-form-text"

  bindings: ->
    @$el.on "submit", @onSubmit

  clearForm: =>
    @$refs.textInput.val("")

  onSubmit: (e) =>
    e.preventDefault()
    $.ajax
      url: @$el.attr("action")
      dataType: "json"
      type: @$el.attr("method")
      data: @$el.serialize()
      success: (comment) =>
        @clearForm()
        @props.onSubmit(comment)
```

```coffee
# app/assets/javascripts/components/comments/list.coffee
class App.CommentsList extends App.Component
  addComment: (comment) =>
    @$el.prepend(@renderComment(comment))

  renderComment: (comment) ->
    JST["comments/item"](comment: comment)
```
