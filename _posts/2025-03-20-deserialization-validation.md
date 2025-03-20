---
layout: single
title:  "Understanding Deserialization in DRF 1 - Validation"
typora-root-url: ../
categories: django
tag: [python, django]
toc: true
author_profile: false
sidebar:
    nav: "docs"
classes: wide
---


## Deserialization Overview 

**Deserialization** is the reverse process of serialization. It involves converting primitive data types (like dictionaries, lists) back into complex data structures (like Django models). In Django REST Framework (DRF), deserialization is typically used to validate and save data from incoming requests.

### Basic Deserialization Example

Consider a example from [official documentation](https://www.django-rest-framework.org/api-guide/serializers/#deserializing-objects) with a `Comment` class and its `CommentSerializer`. 


```python
from rest_framework import serializers
from datetime import datetime

class Comment:
    def __init__(self, email, content, created=None):
        self.email = email
        self.content = content
        self.created = created or datetime.now()

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField() 

# serialize the comment
comment = Comment(email='leila@example.com', content='foo bar')
comment_serializer = CommentSerializer(comment) # pass the instance only
print(f"Comment serialized: \n{comment_serializer.data}")
'''
Comment serialized: {
    'email': 'leila@example.com', 
    'content': 'foo bar', 
    'created': '2025-03-20T07:52:38.562190Z'
}
'''

# deserialize the comment
comment_data = {
    'email': 'leila@example.com',
    'content': 'foo bar',
    'created': datetime.now().isoformat()
}
comment_serializer = CommentSerializer(data=comment_data) # pass the data !
if comment_serializer.is_valid():
    print(f"Comment deserialized: \n{comment_serializer.validated_data}")

'''
Comment deserialized: {
    'email': 'leila@example.com', 
    'content': 'foo bar', 
    'created': datetime.datetime(2025, 3, 20, 7, 52, 38, 562845, tzinfo=zoneinfo.ZoneInfo(key='UTC'))
}
'''
```

There are a few important points to highlight:

1. The deserialization process is not yet complete.  
    The Python primitive dictionary `comment_data` has not been fully converted into a `Comment` instance. We will address the remaining steps of saving the instance from the `validated_data` later.

2. The role of the `is_valid()` method  
    It is necessary to call the `is_valid()` method before accessing the `validated_data` or `data`. This method performs validation during the deserialization process. We will explore the details of the validation process in this post.


### Basic Workflow

1. Creating a serializer instance  
    + Create a serializer instance with the `data` argument.  

    ```python
    comment_serializer = CommentSerializer(data=comment_data)
    ```
2. Validation of the provided data with `is_valid()`

    + When the `is_valid()` method is called, the serializer runs the validation process and saves the result in the `validated_data` attribute. If the validation fails, the `_errors` attribute will be set based on the error message that occurred.

    ```python
    if comment_serializer.is_valid():
        print(f"Comment deserialized: \n{comment_serializer.validated_data}")
    ```

3. Saving the instance using `save()`
    - If an `instance` was also passed when creating the serializer instance, the deserialization process uses the `update` method to update the passed instance using the `validated_data` (Details depend on the implementation of the `update` method).
    - If only the `data` was passed, the deserialization process will create a new instance (Details depend on the implementation of the `create` method).
    ```python
    comment_serializer.save() 
    ```

**In this post, we will focus solely on the validation process.**


## Understanding DRF Serializer Validation

Before we delve into the core of the validation process, let's see what validation functionality DRF provides.


### Field Level Validation
```python
from rest_framework import serializers

class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField() 

    def validate_content(self, value):
        """
        Ensure the content field has a length of less than 10 characters
        """
        if len(value) >= 10:
            raise serializers.ValidationError("Content must be less than 10 characters")
        return value   

# deserialize the comment
comment_data = {
    'email': 'leila@example.com',
    'content': 'foo barasdfasfasfs', # too long (invalid)
    'created': datetime.now().isoformat() 
}
comment_serializer = CommentSerializer(data=comment_data)
if comment_serializer.is_valid():
    print(f"Comment deserialized: \n{comment_serializer.validated_data}")
else: # validation fails
    print(f"Comment is invalid: \n{comment_serializer.errors}")

'''
Comment is invalid: 
{'content': [ErrorDetail(string='Content must be less than 10 characters', code='invalid')]}
'''
```
By implementing functions named `validate_<fieldname>`, we can apply validation rules to each field individually. If a rule is violated, a `ValidationError` should be raised. In the example above, the content field is required to be less than 10 characters long. For more details, refer to [**Field-level validations**](https://www.django-rest-framework.org/api-guide/serializers/#field-level-validation).


---
### Object Level Validation

```python
class CommentSerializer(serializers.Serializer):
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField() 
    last_updated = serializers.DateTimeField()

    def validate(self, attrs):
        """
        Ensure the created field is less than the last_updated field
        """
        print(f"attrs.keys(): {attrs.keys()}")
        # attrs.keys(): dict_keys(['content', 'created', 'last_updated'])
        
        if attrs['created'] > attrs['last_updated']:
            raise serializers.ValidationError("created must be less than last_updated")
        return attrs


# invalid comment data  
comment_data = {
    'content': 'foo barasdfasfasfs', 
    'created': datetime.now()  + timedelta(days=1),
    'last_updated': datetime.now() ,
    'created_at': datetime.now().isoformat()
}
comment_serializer = CommentSerializer(data=comment_data)
if comment_serializer.is_valid():
    print(f"Comment deserialized: \n{comment_serializer.validated_data}")
else:
    print(f"Comment is invalid: \n{comment_serializer.errors}")
'''
Comment is invalid: 
{'non_field_errors': [ErrorDetail(string='created must be less than last_updated', code='invalid')]}
'''
```

We can enforce Object-level validation by overriding the `validate()` method. Similar to how Field-level validation is implemented, it should raise a `ValidationError` when the validation rule is violated. Check out [**Object-level validations**](https://www.django-rest-framework.org/api-guide/serializers/#object-level-validation)




---
### Validators


When enforcing custom validation rules, you don't normally have to use validators. Instead, use field-level and object-level validations. Validators are often used in `ModelSerializer` or for complex customizations. We will handle `Validators` when we deal with `ModelSerializer` later. You can see how it works at [**Validators**](https://www.django-rest-framework.org/api-guide/validators/)



## Validation Core


Now that we've seen how validation works out of the box, it's time to delve deeper and understand its inner workings. Below are the example models we will be working with.


These methods are the core logic of validation. 

- `is_valid()`: This is the primary method for validation, called by the end user.
- `run_validation()`
    - `to_internal_value()` ( where field-level validation occurs)
    - `run_validators()` (where validators are applied)
    - `validate()` ( where object-level validation occurs )





---

### `is_valid()`

```python
# rest_framework/serializers.py
class BaseSerializer(Field):

    def __init__(self, instance=None, data=empty, **kwargs):
        self.instance = instance
        if data is not empty:
            self.initial_data = data
        # omit..
```
When the `data` argument is provided, it is stored in `self.initial_data`.

```python
# rest_framework/serializers.py
class BaseSerializer(Field):

    def is_valid(self, *, raise_exception=False):
        assert hasattr(self, 'initial_data'), (
            'Cannot call `.is_valid()` as no `data=` keyword argument was '
            'passed when instantiating the serializer instance.'
        ) 

        if not hasattr(self, '_validated_data'):
            try:
                self._validated_data = self.run_validation(self.initial_data)
            except ValidationError as exc:
                self._validated_data = {}
                self._errors = exc.detail
            else:
                self._errors = {}

        if self._errors and raise_exception:
            raise ValidationError(self.errors)

        return not bool(self._errors)
```
A few things to note:
- The `is_valid` method assumes that the `data` argument is provided during initialization.
- If validation succeeds, the validated data is saved in `_validated_data` and the method returns `True`.
- If validation fails, the error message is saved in `_errors` and the method returns `False`.


### `run_validation()`

```python
# rest_framework/serializers.py
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):

    def run_validation(self, data=empty):
        # core logic only
        value = self.to_internal_value(data) # field-level validation
        try:
            self.run_validators(value) # validators are applied
            value = self.validate(value) # object-level validation 
            assert value is not None, '.validate() should return the validated data'
        except (ValidationError, DjangoValidationError) as exc:
            raise ValidationError(detail=as_serializer_error(exc))
        return value
```
The `run_validation` method validates data and returns `validated_data` if successful. It uses three key methods:

- `to_internal_value()`: Converts data into a native format with field-level validations and validators, similar to `to_representation()` in serialization.
- `run_validators()`: Runs class-level validators.
- `validate()`: Handles object-level validation, which can be customized by overriding this method. By default, it does nothing.

Let's delve into each method.

### `to_internal_value()`

```python
# rest_framework/serializers.py
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):

    def to_internal_value(self, data):
        """
        Dict of native values <- Dict of primitive datatypes.
        core logic only
        """

        ret = {}
        errors = {}
        fields = self._writable_fields

        for field in fields:

            # retrieve the field-level validation method by its name
            validate_method = getattr(self, 'validate_' + field.field_name, None)
            # get its original data ()
            primitive_value = field.get_value(data)

            try:

                # apply run_validation recursively,
                # for nested-deserialization as with nested-serialization
                # field declared validators are applied in it
                validated_value = field.run_validation(primitive_value)

                # apply field-level validation
                if validate_method is not None:
                    validated_value = validate_method(validated_value)
            except ValidationError as exc:
                errors[field.field_name] = exc.detail
            except DjangoValidationError as exc:
                errors[field.field_name] = get_error_detail(exc)
            except SkipField:
                pass
            else:
                self.set_value(ret, field.source_attrs, validated_value)

        if errors:
            raise ValidationError(errors)
        return ret
```


Key points:

- Retrieves and applies the field-level validation method for each field to the primitive value.
- Calls the `run_validation` method of each field, enabling nested deserialization and invoking field validators.
- Applies field-level validation methods to each field.

### `run_validators()`
```python
# rest_framework/fields.py
class Field:
    def run_validators(self, value):
        # core logic only
        errors = []
        for validator in self.validators:
            try:
                if getattr(validator, 'requires_context', False):
                    validator(value, self)
                else:
                    validator(value)
            except ValidationError as exc:
                # If the validation error contains a mapping of fields to
                # errors then simply raise it immediately rather than
                # attempting to accumulate a list of errors.
                if isinstance(exc.detail, dict):
                    raise
                errors.extend(exc.detail)
            except DjangoValidationError as exc:
                errors.extend(get_error_detail(exc))
        if errors:
            raise ValidationError(errors)


# rest_framework/serializers.py
class Serializer(BaseSerializer, metaclass=SerializerMetaclass):
    def run_validators(self, value):
        # core logic only
        super().run_validators(to_validate)
```
First, it applies default values to read-only fields. Then, it applies class-level validators by calling `Field.run_validators()`. (Note that `Serializer` itself is also a subclass of `Field`)




## References

+ [**Serializers**](https://www.django-rest-framework.org/api-guide/serializers/)
+ [**Validators**](https://www.django-rest-framework.org/api-guide/validators/)
+ [**rest_framework source**](https://github.com/encode/django-rest-framework/tree/master/rest_framework)