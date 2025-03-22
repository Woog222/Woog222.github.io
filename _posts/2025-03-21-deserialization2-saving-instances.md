---
layout: single
title:  "Understanding Deserialization in DRF 2 - Saving Instances"
typora-root-url: ../
categories: django
tag: [python, django]
toc: true
author_profile: false
sidebar:
    nav: "docs"
classes: wide
---




In the previous post, we discussed the validation step in deserialization. If you haven't read it yet, you can check it out [here](../deserialization1-validation/).

Now, let's move on to the next step: saving instances from the validated data.


## Saving Instances 

### Examples

```python
from rest_framework import serializers

class Comment:
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)

    # These method needs to be overridden
    def create(self, validated_data):
        return Comment(**validated_data)

    def update(self, instance, validated_data):
        instance.content = validated_data.get('content', instance.content)
        instance.email = validated_data.get('email', instance.email)
        return instance

# 1. Deserialization - creating a new comment
comment_data = {
    'content': 'Created content', 
    'email': 'test@example.com'
}
comment_serializer = CommentSerializer(data=comment_data)
if comment_serializer.is_valid():
    created_comment = comment_serializer.save()
    print(created_comment.__dict__)
    # {'email': 'test@example.com', 'content': 'Created content'}


# 2. Deserialization - updating an existing comment
new_comment_data = {
    'content': 'Updated content', 
    'email': 'test@example.com'
}
comment_serializer = CommentSerializer(instance=created_comment, data=new_comment_data)
if comment_serializer.is_valid():
    updated_comment = comment_serializer.save()
    print(updated_comment.__dict__)
    # {'email': 'test@example.com', 'content': 'Updated content'}
```
This example demonstrates the basic process of deserializing a Python dictionary into a `Comment` instance. By reading the code, you can easily understand the steps involved. Notice how the `save()` method is responsible for converting the validated data back into a `Comment` instance. Let's examine it further to understand the internal workings.

---


### `save()`

```python
# rest_framework/serializers.py
class BaseSerializer(Field):

    def save(self, **kwargs):
        # CORE LOGIC ONLY

        # Validation Check 1
        assert hasattr(self, '_errors'), (
            'You must call `.is_valid()` before calling `.save()`.'
        )

        # Validation Check 2
        assert not self.errors, (
            'You cannot call `.save()` on a serializer with invalid data.'
        )

        # Data Access Restriction 3
        assert not hasattr(self, '_data'), (
            "You cannot call `.save()` after accessing `serializer.data`."
            "If you need to access data before committing to the database then "
            "inspect 'serializer.validated_data' instead. "
        )

        # Additional Parameters 4
        validated_data = {**self.validated_data, **kwargs}

        # Instance Handling 5
        if self.instance is not None:
            self.instance = self.update(self.instance, validated_data)
            assert self.instance is not None, (
                '`update()` did not return an object instance.'
            )
        else:
            self.instance = self.create(validated_data)
            assert self.instance is not None, (
                '`create()` did not return an object instance.'
            )

        return self.instance
```

The `save()` method either updates an existing instance or creates a new one, and then returns it. Let's examine it further:

+ **Validation Check 1**
    - `self._errors` is not initialized until `self.is_valid()` is called.
    - This ensures that the validations must precede the `save()` method.
+ **Validation Check 2**
    - `self.errors` is a *property* that returns `self._errors`.
    - This ensures that `self._errors` is an empty dictionary, indicating that validation was successful.
+ **Data Access Restriction 3**
    - `self._data` is set when the `self.data` *property* is accessed.
    - This ensures that the `self.data` *property* is not accessed before `save()`.
    - This restriction is likely intended to prevent `self.data` from containing uncommitted data, as the `save()` method is expected to update the database.
    - <u>(If you know the intention behind this, please let me know)</u>
+ **Additional Parameters 4**
    - The `save()` method can accept additional keyword arguments, which are used when saving the instance.
+ **Instance Handling 5**
    - If the serializer has an existing instance, it updates the instance using the `self.update()` method with `validated_data`.
    - Otherwise, it creates a new instance using the `self.create()` method.
    - These methods need to be overridden to implement deserialization.



--- 

### `create()` & `update()`

The implementations of these methods may vary, but the fundamental logic remains simple. (See the function docstring for details)

- `create()`
    - Creates a new instance using the provided `validated_data` and returns it.
    - Performs any additional required actions (e.g., saving the model instance to the database).
- `update()`
    - Updates the existing instance using the provided `validated_data` and returns it.
    - Similarly, performs any additional required actions.

The `Serializer` class does not provide implementations for the `create()` and `update()` methods. This is because, unlike serialization, there are some ambiguous points when dealing with nested deserialization. See this for detail. [**writable-nested-representations**](https://www.django-rest-framework.org/api-guide/serializers/#writable-nested-representations)





## Next Steps - `ModelSerializer`

Now that we understand how `Serializer` works, we can serialize and deserialize any class objects. However, in Django projects, we usually deal with model instances. For each model, we need to define a `Serializer` class specifying the fields and validators to enforce database constraints. Fortunately, DRF provides the `ModelSerializer` class for this purpose. With our understanding of `Serializer`, let's explore `ModelSerializer` and learn how to customize it further.

## References
- [**Serializer docs**]([**Serializers**](https://www.django-rest-framework.org/api-guide/serializers/))
- [**serializers.py**](https://github.com/encode/django-rest-framework/blob/master/rest_framework/serializers.py)