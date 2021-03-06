# golymer

[![Join the chat at https://gitter.im/iheartradio/vclgenie](https://badges.gitter.im/iheartradio/vclgenie.svg)](https://gitter.im/golymer/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

<p align="center">
  <img style="float: right;" src="https://raw.githubusercontent.com/microo8/golymer/master/golypher.png" alt="golymer logo"/>
</p>

Create HTML [custom elements](https://www.w3.org/TR/custom-elements/#custom-element) with [go](https://golang.org) ([gopherjs](https://github.com/gopherjs/gopherjs))

With golymer you can create your own HTML custom elements, just by registering a go struct. The content of the `shadowDOM` has automatic data bindings to the struct fields. Read an [blog post](https://medium.com/@magyarvladimir/web-components-in-go-3a2488725f68).

Contribution of all kind is welcome. Tips for improvement or api simplification also :)


```go
package main

import "github.com/microo8/golymer"

var myAwesomeTemplate = golymer.NewTemplate(`
<style>
	:host {
		background-color: blue;
		width: 500px;
		height: [[FooAttr]]px;
	}
</style>
<p>[[privateProperty]]</p>
<input type="text" value="{{BarAttr}}"/>
`)

type MyAwesomeElement struct {
	golymer.Element
	FooAttr int
	BarAttr string
	privateProperty float64
}

func NewMyAwesomeElement() *MyAwesomeElement {
	e := &MyAwesomeElement{
		FooAttr: 800,
	}
	e.SetTemplate(myAwesomeTemplate)
	return e
}

func main() {
	//pass the element constructor to the Define function
	err := golymer.Define(NewMyAwesomeElement)
	if err != nil {
		panic(err)
	}
}
```

Then just run `$ gopherjs build`, import the generated script to your html `<script src="my_awesome_element.js"></script>` and you can use your new element

```html
<my-awesome-element foo-attr="1" bar-attr="hello"></my-awesome-element>
```

## define an element

To define your own custom element, you must create an struct that embeds the `golymer.Element` struct and a function that is the constructor for the struct. Then add the constructor to the `golymer.Define` function. This is an minimal example.:

```go
type MyElem struct {
	golymer.Element
}

func NewMyElem() *MyElem {
	return new(MyElem)
}

func init() {
	err := golymer.Define(NewMyElem)
	if err != nil {
		panic(err)
	}
}
```

The struct name, in CamelCase, is converted to the kebab-case. Because html custom elements must have at least one dash in the name, the struct name must also have at least one "hump" in the camel case name. `(MyElem -> my-elem)`. So, for example, an struct named `Foo` will not be defined and the `Define` function will return an error.

Also the constructor fuction must have a special shape. It can't take no arguments and must return a *pointer* to an struct that embeds the `golymer.Element` struct.

## element attributes

golymer creates attributes on the custom element from the exported fields and syncs the values.

```go
type MyElem struct {
	golymer.Element
	ExportedField string
	unexportedField int
	Foo float64
	Bar bool
}
```

Exported fields have attributes on the element. This enables to declaratively set the api of the new element. The attributes are also converted to kebab-case.

```html
<my-elem exported-field="value" foo="3.14" bar="true"></my-elem>
```

HTML element attributes can have just text values, so golymer parses these values to convert it to the field types with the [strconv](https://golang.org/pkg/strconv) package.

## lifecycle callbacks

`golymer.Element` implemets the `golymer.CustomElement` interface. It's an interface for the custom elements lifecycle in the DOM. 

`ConnectedCallback()` called when the element is connected to the DOM. Override this callback for setting some fields, or spin up some goroutines, but remember to call the `golymer.Element` also (`myElem.Element.ConnectedCallback()`).

`DisconnectedCallback()` called when the element is disconnected from the DOM. Use this to release some resources, or stop goroutines.

`AttributeChangedCallback(attributeName string, oldValue string, newValue string, namespace string)` this callback called when an observed attribute value is changed. golymer automatically observes all exported fields. When overriding this, also remember to call `golymer.Element` callback (`myElem.Element.AttributeChangedCallback(attributeName, oldValue, newValue, namespace)`).


## template

The function `golymer.NewTemplate` will create a new [template](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template) element, from which the new custom element's `shadowDOM` will be instantiated. It is better to use the `template` element for this, and not `innerHTML`, because the browser must parse the html only once.

If you want to create an custom element with a template, you must set the elements template in the constructor with the `SetTemplate` function.

```go

var myTemplate = golymer.NewTemplate(`<h1>Hello golymer</h1>`)

func NewMyElem() *MyElem {
	e := new(MyElem)
	e.SetTemplate(myTemplate)
	return e
}
```

The element will then have an `shadowDOM` thats content will be set from the `Template` element at the `connectedCallback`.

## one way data bindings

golymer has build in data bindings. One way data bindings are used for presenting an struct field's value. For defining an one way databinding you can use double square brackets with the path to the field (`[[Field]]` or `[[subObject.Field]]`) Eg:

```html
<p>[[Text]]!!!</p>
```

Where the host struct has an `Text` field. Or the name in brackets can be an path to an fielt `subObject.subSubObject.Field`. The field value is then converted to it's string representation. One way data bindings can be used in text nodes, like in the example above, and also in element attributes eg. `<div style="display: [[MyDisplay]];"></div>`

Every time the fields value is changed, the template will be automaticaly changed. Changing the `Text` fields value eg `myElem.Text = "foo"` also changes the `<p>` element's `innerHTML`.

## two way data bindings

Two way data bindings are declared with two curly brackets (`{{Field}}` or `{{subObject.Field}}`) and work only in attributes of elements in the template. So every time the elements attribute is changed, the declared struct field will also be changed. golymer makes also an workaround for html `input` elements, so it is posible to just bind to the `value` attribute.

```html
<input id="username" name="username" type="text" value="{{Username}}">
```

Changing `elem.Username` changes the `input.value`, and also changing the `input.value` or the value attribute `document.getElementById("username").setAttribute("value", "newValue")` or the user adds some text, the `elem.Username` will be also changed.

It's also possible to pass complex data structures to the sub element with two way data bindings.

```go
type User struct {
	ID   int
	Name string
}

type ElemOne struct {
	golymer.Element
	Usr  *User
	...
}
type ElemTwo struct {
	golymer.Element
	Usr  *User
	...
}
```

`ElemOne` template:

```html
<div>
	<elem-two usr="{{Usr}}"></elem-two>
</div>
```

This will keep the `elemOne.Usr` and `elemTwo.Usr` the same pointer to the same object.

## connecting to events

Connecting to the events of elements can be done with the javascript `addEventListener` function, but it is also possible to connect some struct method with an `on-<eventName>` attribute.  

```html
<button on-click="ButtonClicked"></button>
```

Event handlers get the event as a pointer to `golymer.Event`

```go
func (e *MyElem) ButtonClicked(event *golymer.Event) {
	print("the button was clicked!")
}
```

## custom events

golymer adds the `DispatchEvent` method so you can fire your own events.

```go
event := golymer.NewEvent(
	"my-event",
	map[string]interface{}{
		"detail": map[string]interface{}{
			"data": "foo",
		},
		"bubbles": true,
	},
)
elem.DispatchEvent(event)
```

and these events can be also connected to:

```html
<my-second-element on-my-event="MyCustomHandler"></my-second-element>
```

## observers

On changing an fields value, you can have an observer, that will get the old and new value of the field. It must just be an method with the name: `Observer<FieldName>`. eg:

```go
func (e *MyElem) ObserverText(oldValue, newValue string) {
	print("Text field changed from", oldValue, "to", newValue)
}
```

## children

golymer scans the template and checks the `id` of all elements in it. The `id` will then be used to map the children of the custom element and can be accessed from the `Childen` map (`map[string]*js.Object`). Attribute `id` cannot be databinded (it's value must be constant). 

```go
var myTemplate = golymer.NewTemplate(`
<h1 id="heading">Heading</h1>
<my-second-element id="second"></my-second-element>
<button on-click="Click">click me</button>
`)

func (e *MyElem) Click(event *golymer.Event) {
	secondElem := e.Children["second"].Interface().(*MySecondElement)
	secondElem.DoSomething()
}
```

golymer also populates fields of the custom element from the `id`. If the element's `id` is equal to the field name and if it's type is the same as the element in template. eg:

```go
var tmpl = golymer.NewTemplate(`
<my-second-element id="second"></my-second-element>
`)

type MyElem struct {
	golymer.Element
	second *MySecondElement //this field will after ConnectedCallback have an pointer to the `<my-second-element>` in the template
}
```

## type assertion

It is possible to type assert the node object to your custom struct type. With selecting the node from the DOM directly

```go
myElem := js.Global.Get("document").Call("getElementById", "myElem").Interface().(*MyElem)
```

and also from the `Children` map

```go
secondElem := e.Children["second"].Interface().(*MySecondElement)
```

## element creation

Calling your custom element's constructor creates you just the struct type. The node must be created by the DOM. So you must create your registered element with `document.createElement`:

```go
myElem := js.Global.Get("document").Call("createElement", "my-elem").Interface().(*MyElem)
```

Or you can use the helper function from golymer:

```go
myElem := golymer.CreateElement("my-elem").(*MyElem)
```

# golymer elements

The subpackage [golymer elements](https://github.com/microo8/golymer/tree/master/elements) contains common elemets needed in building websites. 

Like some [material design](https://material.io/) elements and also some common declarative dom manipulation elements.

But it's still an work in progress. The collection isn't complete, but elements that are already complete are functioning and usable.

# SPA

The subpackage [spa](https://github.com/microo8/golymer/tree/master/spa/) contains utils/helpers for building single page applications. 
