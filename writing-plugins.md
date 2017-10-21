---
permalink: /writing-plugins/
title: Writing plugins
---

<div id="leftmenu" class="col-sm-4 col-md-3 hidden-xs">
<ul class="nav nav-list side-navigation" data-spy="affix" data-offset-top="{{ site.affix_offset }}">
    <li><a href="#misc">Don'ts and do's</a></li>
    <li><a href="#example">An example plugin</a></li>
    <li><a href="#render">Renderer plugins</a></li>
</ul>
</div>
<div class="col-sm-8 col-md-9" markdown="1">

# Writing plugins

<section id="misc" markdown="1">

## Don'ts and do's

Kint and its plugins are in the unique situation of taking a variable by reference and being told not to do anything with it. The absolutely worst thing you can ever do is change an input variable.

If you do this you may fubar a user's data. That's a no-no whether you're writing a plugin for inclusion in Kint or for your own system.

**Never alter input data in a plugin. I will find you.**

### Accessing the parser

All plugins have the parser they're assigned to added to them through the `setParser()` method. If you have new data you want parsed into a `Kint_Object` you do so via the parser:

<pre class="prettyprint linenums">
$more_data = get_data_from_somewhere($var);
$base_object = Kint_Object::blank('Magic somewhere data');
$base_object->depth = $o->depth + 1;
$new_object = $this->parser->parse($more_data, $base_object);
</pre>

### Arrays

Recursion detection in arrays is performed by adding a random-generated unique key to the array as a marker when parsing. If the parser sees this marker when parsing an array it knows it's been here before and is recursing.

As a result, when you try to read directly from an input variable that also happens to be an array, you'll end up with extra data. The way to fix this is to call `$this->parser->getCleanArray()` on the variable, which will shallow copy the array and remove the recursion marker for you.

Note that as this is a shallow copy the first rule on this page still applies to the array's contents.

### Halting the parse

If you want to prevent any other plugins or the parser from messing with your carefully crafted data, you'll want to call `$this->parser->haltParse()`. This will stop any more plugins from running and prevent the base parse too if your plugin is running first.

</section>
<section id="example" markdown="1">

## An example plugin

Let's imagine we're working on a system that's a big black box. Everything has an ID, and we can get all the data we need by calling a function for that ID.

Wouldn't it be great if we could automatically show the data associated with an ID whenever we come across one?

### A barebones plugin

<pre class="prettyprint linenums">
<?php

class MyKintParserPlugin extends Kint_Parser_Plugin
{
    public function getTypes()
    {
        return array('integer', 'string');
    }

    public function getTriggers()
    {
        return Kint_Parser::TRIGGER_SUCCESS;
    }

    public function parse(&amp;$var, Kint_Object &amp;$o, $trigger)
    {
        echo 'My parser found: ';
        var_dump($var);
    }
}

Kint::$plugins[] = 'MyKintParserPlugin';

d(1234);
</pre>

Here we can see the 3 required methods of a Kint_Parser_Plugin.

* `getTypes()` returns the types of data this plugin can operate on. These are types as returned by `gettype()`. Since we're taking IDs they will probably be either strings or integers, so we return an array with both types.
* `getTriggers()` returns a bitmask of the events that will trigger this plugin. These are all constants of the `Kint_Parser` class.
    * `TRIGGER_BEGIN` runs before any parsing is done
    * `TRIGGER_SUCCESS` runs after parsing successfully finishes
    * `TRIGGER_DEPTH_LIMIT` and `TRIGGER_RECURSION` run after parsing is halted
    * `TRIGGER_COMPLETE` is `TRIGGER_SUCCESS | TRIGGER_RECURSION | TRIGGER_DEPTH_LIMIT`

    We're going to use `TRIGGER_SUCCESS` for our plugin.
* `parse()` is the workhorse of the plugin. This is what's called when the plugin is expected to alter data. We'll just put a `var_dump` in here to make sure everything's working.

Lastly, we add the plugin to the `Kint::$plugins` array and try it out.

> ![]({{ site.baseurl }}/images/plugin-before.png)

Yay!

### Implementing our plugin's functionality

<pre class="prettyprint linenums:15">
public function parse(&amp;$var, Kint_Object &amp;$o, $trigger)
{
    if (!ctype_digit((string) $var)) {
        return;
    }

    global $big_black_box;
    $data = $big_black_box->get_data_from_id($var);
    if (empty($data)) {
        return;
    }

    $base_object = Kint_Object::blank('Black box data');
    $base_object->depth = $o->depth;

    if ($o->access_path) {
        $base_object->access_path = '$GLOBALS[\'big_black_box\']->get_data_from_id('.$o->access_path.')';
    }

    $r = new Kint_Object_Representation('Black box data');
    $r->contents = $this->parser->parse($data, $base_object);

    $o->addRepresentation($r);
}
</pre>

* Check that what we have is actually an ID. If it's a random string we don't need to waste time trying to get data from it, so we'll just return.
* Get the data we want to add to the dump. If we couldn't find any we'll just return.
* Make our "Base object" - this needs to contain information the parser can't get about the variable like its name, access path, depth, whether it's public or private, etc.
* If we have an access path to the variable we're parsing now, we can continue the access path into the data by wrapping the current one in the code we need to get the data.

    This means if the ID is found at `$array['key']->prop` then `$data['children']` will have an accurate access path like:

    `$GLOBALS['big_black_box']->get_data_from_id($array['key']->prop)['children']`
* Make a new representation and put the parsed data inside it
* Add the representation to the object

> ![]({{ site.baseurl }}/images/plugin-after.png)

Yay!

---

You can look at the source code for the plugins shipped with Kint by default for more detailed examples on various behaviors.

</section>
<section id="render" markdown="1">

## Renderer plugins

Renderers don't have a unified plugin system, it's implemented by the individual renderers at will.

The one common factor is that parser plugins will attach strings to the `hints` array on `Kint_Object` and `Kint_Object_Representation` which the renderer can use to decide what to do without having to re-parse the objects.

In the case of the rich renderer there are 2 separate plugin pools, where the key is the hint and the value is the plugin class:

* `Kint_Renderer_Rich::$object_renderers`: How to render an object itself. This alters both the way the bar for an object appears, and how the children are rendered. For example, it's an object renderer that adds the color swatch to the bar for a color string.
* `Kint_Renderer_Rich::$tab_renderers`: How to render an individual tab. For example, how to render the docstring when you open a method.

You can also write your own renderer from scratch. For an example see the [kint-js project](https://github.com/kint-php/kint-js).

</section>

</div>