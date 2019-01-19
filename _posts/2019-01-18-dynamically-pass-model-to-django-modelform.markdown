---
layout: post
title:  "Dynamically Pass Model to Django ModelForm"
date:   2019-01-18 22:42:50 +0100
categories: Python Django
author: Boni
---

## Context

Let's say you'd like to use Django's powerful `ModelForm` for multiple models &mdash; it goes without saying that if you're trying to create an abstract Django application to handle multiple models within your project, you'd like to define the `ModelForm` only once and not have to create custom `ModelForm`'s for each of the models you'd like to support.

## Concrete scenario

When I was asked to implement revert and recovery features for one of the projects I'm working on I easily decided to use the robust [django-reversion] extension &mdash; the thing is that it only comes with full support for Django Admin, and if you have to use it within your custom views and templates, you have to use the extension's API directly and write your own views and templates.

## Problem

If you want to use the `ModelForm`, you have to define the `model` property on the `Meta` class, e.g.:

{% highlight python %}
# forms.py

from django.forms import ModelForm
from myapp.models import MyModel

# Create the form class.
class MyModelForm(ModelForm):
    class Meta:
        model = MyModel
        fields = ['title', 'date', ...]
{% endhighlight %}

## Solution

The solution is simple if you remember that in Python, functions can return class definitions &mdash; so, within your `forms.py` you can write a form helper function:

{% highlight python %}
# forms.py

from django.forms import ModelForm

def get_model_form(dynamic_model, form_fields='__all__'):
    class MyDynamicModelForm(ModelForm):
        class Meta:
            model = dynamic_model
            fields = form_fields

    return MyDynamicModelForm
{% endhighlight %}

Then, simply use the `get_model_form` helper function in your view to retrieve the `ModelForm`.

If you're using Django class based views (and in this particular example I'm using `FormView`), you can override the `get_form_class` method and return the call to the `get_model_form` function, e.g.:

{% highlight python %}
# views.py

from django.apps import apps
from django.views.generic import FormView

from myapp.forms import get_model_form


class MyDynamicModelView(FormView):
    ...

    def dispatch(self, request, *args, **kwargs):
        # Assuming you're passing `app_label` and `model_name` via URL
        app_label = kwargs['app_label']
        model_name = kwargs['model_name']
        self.model = apps.get_model(app_label, model_name)
        return super().dispatch(request, *args, **kwargs)

    def get_form_class(self):
        return get_model_form(self.model)

    ...
{% endhighlight %}

## Conclusion

To pass model to the `ModelForm` dynamically:
- create a helper function which accepts the custom model as parameter;
- use the call to that helper function when you're supposed to define the Django form class.

[django-reversion]: https://github.com/etianen/django-reversion/
