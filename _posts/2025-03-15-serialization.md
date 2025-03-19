---
layout: single
title:  "Serialization Process"
typora-root-url: ../
categories: django
tag: [python, django]
toc: true
author_profile: false
sidebar:
    nav: "docs"
classes: wide
---





## Serialization

**Serialization** is the process of converting complex data structures (like Django models or querysets) into Python primitive data types (such as dictionaries) that can be easily rendered into JSON, XML, or other content types. In Django REST Framework, this is accomplished by passing an `instance` to a serializer, which then transforms the object's attributes into a dictionary representation suitable for API responses.

When serializing, the serializer's `to_representation` method is called internally to convert each field of the model instance into its corresponding primitive type. This process happens automatically when you access the serializer's `data` property.


### Examples

Let's begin our exploration with straightforward serialization examples to illustrate the core concepts.


#### Single object Serialization 

```python
from datetime import datetime
from rest_framework import serializers

class Comment: # not a djnago model
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content
        self.created = created or datetime.now()

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

```bash

$ python manage.py shell
# omit import

# Serializing a single instance
>>> comment = Comment(email = "abc134@gmail.com", content = "hello!") 
>>> serializer = CommentSerializer(instance=comment) # Pass an instance only
>>> serializer.data 
{'email': 'abc134@gmail.com', 'content': 'hello!', 'created': '2025-03-19T06:58:28.929939Z'}

# Serializing multiple instances
>>> comments = [ Comment(email=f"email{i}@gmail.com", content=f"content{i}") for i in range(1,4) ] 
>>> serializer= CommentSerializer(instance = comments, many=True) # Pass a list of instances instead
>>> serializer.data
[
{'email': 'email1@gmail.com', 'content': 'content1', 'created': '2025-03-19T07:34:08.484335Z'}, 
{'email': 'email2@gmail.com', 'content': 'content2', 'created': '2025-03-19T07:34:08.484341Z'}, 
{'email': 'email3@gmail.com', 'content': 'content3', 'created': '2025-03-19T07:34:08.484342Z'}
]
```

#### Single django model Serialization

```python
from django.db import models
from rest_framework import serializers

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=200)

class BookSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    author = serializers.CharField(max_length=200)
```

```bash
$ python manage.py shell
# omit import
>>> for i in range(1,4): Book(title=f"title{i}", author=f"author{i}").save() # Save 3 instances into DB

# Serializing a single instance
>>> serializer = BookSerializer(instance= queryset[0])
>>> serializer.data
{'id': 1, 'title': 'title1', 'author': 'author1'}

# Serializing multiple instances
>>> queryset = Book.objects.all()
>>> serializer = BookSerializer(instance=queryset, many=True) # Pass the queryset instead, with many = True
>>> serializer.data
[
{'id': 1, 'title': 'title1', 'author': 'author1'}, 
{'id': 2, 'title': 'title2', 'author': 'author2'}, 
{'id': 3, 'title': 'title3', 'author': 'author3'}
]
```



## Core 

### Base model

```python
from django.db import models
from rest_framework import serializers

class Person(models.Model):
    name = models.CharField(max_length=200)
    age = models.IntegerField()
    email = models.EmailField(blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)

class PersonSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=200)
    age = serializers.IntegerField()
    

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Person, on_delete=models.CASCADE, related_name='simple_books')
    published = models.BooleanField(default=False)

    def __str__(self):
        return f"{title} written by {author}, published at {published}"


class BookSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    author = PersonSerializer(read_only=True) # The Serializer itself is also a subclass of Field
    published = serializers.BooleanField()
```

Now, let's explore the internal flow of the serialization process in depth by examining the source code. We'll use the ```Book``` model and its ```BookSerializer``` as examples to illustrate how serialization works under the hood.

```python
author = Person.objects.create(name="John Doe", age=30, email="john.doe@example.com")
book = Book.objects.create(title="My First Book", author=author, published=True)
book_serializer = BookSerializer(book) 
book_serializer.data
'''
{
    'id': 1,    
    'title': 'My First Book', 
    'author': {
        'id': 1,    
        'name': 'John Doe', 
        'age': 30
    }, 
    'published': True
}
'''
```

This is the result of serialization. Notice an important detail: the `PersonSerializer` class itself is declared as one of the serializer fields in `BookSerializer`. This field handles the serialization of the `Person` model, which is a foreign key in the `Book` model. This pattern is known as *nested serialization*. Let's examine how this process works internally. 




### ```BaseSerializer``` 

[**serializers.py**](https://github.com/encode/django-rest-framework/blob/master/rest_framework/serializers.py#)

```python
author = Person.objects.create(name="John Doe", age=30, email="john.doe@example.com")
book = Book.objects.create(title="My First Book", author=author, published=True)
book_serializer = BookSerializer(book) 

print(book_serializer.instance)
# My First Book written by John Doe (30 years old), published at True
```

Now that we've created a ```BookSerializer``` instance with only a book instance passed to its initializer, the ```book_serializer``` stores this book object in its ```instance``` attribute. This is the starting point for the serialization process.

This is how the initializer of ```Serializer``` class looks like

```python
# rest_framework/serializers.py
class BaseSerializer(Field):
    def __init__(self, instance=None, data=empty, **kwargs):
        self.instance = instance 
        if data is not empty:
            self.initial_data = data
        self.partial = kwargs.pop('partial', False)
        self._context = kwargs.pop('context', {})
        kwargs.pop('many', None)
        super().__init__(**kwargs)
```


When we access ```serializer.data``` to retrieve the serialization result, it internally calls the ```to_representation``` method. The ```to_representation``` method is a crucial part of the serialization process, as it handles the conversion of object instances into Python primitive data types.
```python
book_serializer.data
'''
{
    'id': 1,    
    'title': 'My First Book', 
    'author': {
        'id': 1,    
        'name': 'John Doe', 
        'age': 30
    }, 
    'published': True
}
'''
```

```python
# rest_framework/serializers.py
class BaseSerializer(Field):
    @property
    def data(self):
        # omit details
        if not hasattr(self, '_data'):
            if self.instance is not None and not getattr(self, '_errors', None):
                self._data = self.to_representation(self.instance) # This is the case of serialization
            elif hasattr(self, '_validated_data') and not getattr(self, '_errors', None):
                self._data = self.to_representation(self.validated_data)
            else:
                self._data = self.get_initial()
        return self._data
```
The `to_representation` method is key to serialization in both `Serializer` and `Field` classes. Since `Serializer` inherits from `Field`, this method serves the same purpose in both: converting complex objects into primitive dictionaries that can be easily serialized to JSON. We'll explore this inheritance pattern further when discussing nested serialization.

### ```to_representation```

[**serializers.py**](https://github.com/encode/django-rest-framework/blob/master/rest_framework/serializers.py#L518)  
[**fields.py**](https://github.com/encode/django-rest-framework/blob/master/rest_framework/fields.py)

```python
# rest_framework/serializers.py
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):
    def to_representation(self, instance):
        """
        Object instance(Book object) -> Dict of primitive datatypes. 
        Code shown here is simplified for clarity
        """
        ret = {}
        fields = self._readable_fields 

        for field in fields:
            try:
                attribute = field.get_attribute(instance)
            except SkipField:
                continue

            ret[field.field_name] = field.to_representation(attribute)
        return ret
```
```self._readable_fields``` is a Python generator that yields the declared field objects in the serializer class. For more details, refer to the serializers.py source code. For those who are curious about the method's inner workings, see below.

```python
for field in book_serializer._readable_fields:
    print(f"field : {field}")
    print(f"field_name : {field.field_name}")
    attribute = field.get_attribute(book) # instance 
    print(f"attribute : {attribute}")
    value = field.to_representation(attribute)
    print(f"value : {value}", end="\n\n")

'''
field : IntegerField(read_only=True)
field_name : id
attribute : 1
value : 1

field : CharField(max_length=200)
field_name : title
attribute : My First Book
value : My First Book

field : PersonSerializer(read_only=True):
    id = IntegerField(read_only=True)
    name = CharField(max_length=200)
    age = IntegerField()
field_name : author
attribute : John Doe (30 years old)
value : {'id': 1, 'name': 'John Doe', 'age': 30}

field : BooleanField()
field_name : published
attribute : True
value : True
'''
```

- `Field.field_name` represents the variable name declared in the serializer.
- `Field.get_attribute` extracts the corresponding value of the instance. (see [`Field.get_attribute()`](https://github.com/encode/django-rest-framework/blob/master/rest_framework/fields.py#L433))
- `Field.to_representation` transforms its attribute from `get_attribute` according to the format specified by the corresponding Field class.

Note that `PersonSerializer` is one of the fields being serialized. It recursively calls its `to_representation` method for nested serialization. Do you now understand the concept of nested serialization and the well-structured system of `Field` and `Serializer` classes? Can you now see how the serialized result below is generated?

```python
author = Person.objects.create(name="John Doe", age=30, email="john.doe@example.com")
book = Book.objects.create(title="My First Book", author=author, published=True)
book_serializer = BookSerializer(book) 
book_serializer.data
'''
{
    'id': 1,    
    'title': 'My First Book', 
    'author': {
        'id': 1,    
        'name': 'John Doe', 
        'age': 30
    }, 
    'published': True
}
'''
```



## References

- [**Serializers documentation**](https://www.django-rest-framework.org/api-guide/serializers/)
- [**django-rest-framework source**](https://github.com/encode/django-rest-framework/tree/master)