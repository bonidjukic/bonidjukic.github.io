---
layout: post
title:  "Serialize Django Model Class Properties"
date:   2019-02-04 21:37:50 +0100
categories: Python Django
author: Boni
---

## Context

Using [Django's serialization framework] to serialize data into the most popular formats, i.e. `xml`, `json` or `yaml` is quite straightforward. This can be quite useful if you're e.g. trying to pass some json encoded data to the client-side and you don't really want to add the entire [django-rest-framework] to your list of dependencies.

## Concrete scenario

I'm often using [select2] `jQuery` component in my Django projects when I want a powerful yet simple autocomplete select widgets. If I just need to render a mostly static list of options for the select autocomplete, most of the time I find it easiest to simply pass a `json` encoded `QuerySet` and initialize the `select2` component with the passed `json` object.

## Problem

Using the built-in serializers from the `core` module is pretty easy:

{% highlight python %}
from django.core import serializers

data = serializers.serialize('json', MyModel.objects.all())
{% endhighlight %}

And even though you can pass the `fields` argument to only serialize a specified subset of the model fields:

{% highlight python %}
from django.core import serializers

data = serializers.serialize('json', MyModel.objects.all(),
                             fields=('name', 'description'))
{% endhighlight %}

**It is not possible to pass the list of model class properties:**

{% highlight python %}
from django.core import serializers

data = serializers.serialize('json', MyModel.objects.all(),
                             fields=('name', 'description'),
                             props=('display_name',)) # not supported, yet!
{% endhighlight %}

## Solution

The solution is to write a custom `Serializer`, inheriting from a couple of Django core serializers to make things DRY:

{% highlight python %}
# serializers.py

from django.core import serializers


class PropBaseSerializer(serializers.base.Serializer):
    """
    Custom serializer class which enables us to specify a subset
    of model class properties (as well as fields)
    """
    def serialize(self, queryset, **options):
        self.selected_props = options.pop('props')
        return super().serialize(queryset, **options)

    def serialize_property(self, obj):
        model = type(obj)
        for prop in self.selected_props:
            if hasattr(model, prop) and type(getattr(model, prop)) == property:
                self.handle_prop(obj, prop)

    def handle_prop(self, obj, prop):
        self._current[prop] = getattr(obj, prop)

    def end_object(self, obj):
        self.serialize_property(obj)
        super().end_object(obj)


class PropPythonSerializer(PropBaseSerializer, serializers.python.Serializer):
    pass


class PropJsonSerializer(PropPythonSerializer, serializers.json.Serializer):
    pass
{% endhighlight %}

The usage is now straightforward, e.g. in your views:

{% highlight python %}
# views.py

from myapp.serializers import PropJsonSerializer


PropJsonSerializer().serialize(MyModel.objects.all(),
                               fields=('name', 'description'),
                               props=('display_name',))
{% endhighlight %}

## Adding a bit shorter wrapper function

I usually add a helper function to the `serializers.py` as well to have a bit more concise serialize step:

{% highlight python %}
# serializers.py

def json_serialize(qs, fields=(), props=()):
    return PropJsonSerializer().serialize(qs, fields=fields, props=props)
{% endhighlight %}

And now you can use it within your views e.g.:

{% highlight python %}
# serializers.py

from apps.core.serializers import json_serialize


data = json_serialize(MyModel.objects.all(), fields=('name', 'description'),
                      props=('display_name',))
{% endhighlight %}

Or, not passing fields at all:

{% highlight python %}
# serializers.py

from apps.core.serializers import json_serialize


data = json_serialize(MyModel.objects.all(), props=('display_name',))
{% endhighlight %}

## Conclusion

To include model class properties in the serialization output:
- write a custom `Serializer` class within your `serializers.py`;
- use that serializer to pass the `props` argument to specify the subset of class properties to serialize;
- optionally, write a bit shorter wrapper function to save a few letters when serializing data.

[django-rest-framework]: https://www.django-rest-framework.org/
[Django's serialization framework]: https://docs.djangoproject.com/en/dev/topics/serialization/
[select2]: https://github.com/select2/select2
