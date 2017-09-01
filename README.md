# sequelize-to-json

A fast, simple JSON serializer for your [Sequelize](https://github.com/sequelize/sequelize) models. Turns model instances into plain JSON-friendly objects that can be safely embedded in API responses. Suitable for serializing large numbers of instances at once.

Serialization can be performed according to named _schemes_, of which any number can be defined for a model. A scheme specifies the properties to be be included or excluded in the result, and can also provide schemes to use for associated models.

_Should_ work with both Sequelize 3.x and 4.x. Examples provided in this document are based on Sequelize 3. See [here](http://docs.sequelizejs.com/manual/tutorial/upgrade-to-v4.html) for the list of changes introduced in the new major release of Sequelize.

Proper test suite is the next thing on the roadmap. Until it comes into existence the package shoud not be considered stable.

`sequelize-to-json` requires Node 4.4.x or later.

## Installation

Install it within your project's tree using NPM: 

`npm install --save sequelize-to-json`

The module doesn't install any dependiences. It just ships with a custom minimized `lodash` build (to be removed in future releases).

## Usage

A nice walkthrough has yet to be written. For now, just a crude example.

Assuming we already have a working Sequelize instance providing us with `User` and `BlogPost` models and at least the `author` relation:

```js
const 
  Serializer = require('sequelize-to-json'),
  db = require('./db'), // our Sequelize instance, with models and relations already defined
  BlogPost = db.model('BlogPost'),
  User = db.model('User');

// our serialization scheme for the `BlogPost` model with the associated `User` (stored in the `author` field)
const scheme = {
  // include all own properties and the associated `User` instance
  include: ['@all', 'author'],
  // let's exclude from the above the primary key and all foreign keys
  exclude: ['@pk', '@fk'],
  assoc: {
    // scheme to be used for the associated `User` instance
    author: {
      // include just a selection of fields (URLs are method calls but they could be implemented as VIRTUAL attributes as well)
      include: ['fullName', 'aboutMe', 'getProfileUrl', 'getAvatarUrl'],
      // let's assign better names to properties obtained via instance methods
      as: { getProfileUrl: 'profileUrl', getAvatarUrl: 'avatarUrl' }
    }
  }
};
  
// now fetch some posts, with authors, and serialize them
BlogPost.findAll({
  include: { model: User, as: 'author' },
  // ...
}).then(function(posts) {
  // serialize all the items efficiently
  let postsAsJSON = Serializer.serializeMany(posts, BlogPost, scheme);
  
  // serialize just the first item
  let serializer = new Serializer(BlogPost, scheme);
  let postAsJSON = serializer.serialize(posts[0]);
  // ...
});
```

Resulting JSONified post objects would look like this (assuming `BlogPost` contains the `title` and `content` fields):

```js
{
  "title": ...,
  "content": ...,
  ...,
  "author": {
    "fullName": ...,
    "aboutMe": ...,
    "profileUrl": ...,
    "avatarUrl": ...
  }
}
```

See the [Reference](#reference) below for the complete description of `sequelize-to-json` features.

## Reference

### The `Serializer` class

Represents a serializer. 

Serializer objects can process any number of instances of a given model using the specified scheme. Both the model class and the scheme are tied to a serializer object; you'll need to instantiate a new serializer for each model / scheme pair. 

This class is the only thing exported by the module. It can be accessed directly by requiring `sequelize-to-json`. 

(The class doesn't actually have a name of it's own; i'm using here the name `Serializer` just for clarity. You can import it under any name you wish.)

<a id="serializer-serializer"></a>
#### `Serializer(model, [scheme, [options]])`


Creates a new serializer.

Arguments are:

* **`model`** - Sequelize model class whose instances are to be serialized
* **`scheme`** - scheme to be used for serialization. Can be an object, a string or anything falsy.
* **`options`** - object containing additional options (see [Options](#options))

If `scheme` is a string, the scheme definition will be searched for in `model.serializer.schemes`. If `scheme` is falsy, following steps will be performed to identify the serialization scheme:

1. If `model.serializer.defaultScheme` is set, it will be interpreted as the name of scheme to use (from `model.serializer.schemes`).
2. If there's no `model.serializer.defaultScheme`, the scheme is checked for in `model.serializer.schemes.default`.
3. If there's no scheme named `default` in `model.serializer.schemes`, the global default scheme is used (see below).

For info on defining schemes, see the [schemes section](#defining-schemes).

<a id="serializer-serialize"></a>
#### `#serialize(instance, [cacheObj])`

Serializes a single model instance into a JSON-friendly object, according to the scheme provided in constructor.

**`instance`** must be an instance of the proper model class, otherwise the method will throw an error.

**`cacheObj`** should only be used if the `serialize` method is to be called repeatedly, eg. when processing a collection of model instances. In such cases, the `cacheObj` argument should be a reference to an existing plain object, which will serve as a cache for storing `Serializer` instances created recursively to process associated model instances.

For example:

```js
let serializer = new Serializer(SomeModel, someScheme);
let cache = {}, json = [];

for(let item of items) {
  json.push(serializer.serialize(item, cache));
}
```

Without passing `cache` in the `.serialize` call (or with `cache` being set inline to `{}`!) the performance will drop significantly since `Serializer` objects will have to be re-created for each item.

<a id="serializer-serialize-many"></a>
#### `.serializeMany(instances, model, [scheme, [options]])`

Convenience static method for serializing collections of model instances. It uses the `Serializer` object cache internally so that serializers are not re-created for associated model instances. See [`#serialize()`](#serializer-serialize) for details.

The `model`, `scheme` and `options` parameters are passed to the `Serializer` constructor.

<a id="serializer-encode-to-json"></a>
#### `.encodeToJSON(obj, [options])`

Converts regular Javascript objects into JSON-friendly format. This function is used by default for the conversion of values returned by Sequelize.

**`obj`** is the object (value) to be encoded. **`options`** can be used to fine-tune the encoder.

Currently supported options:

* **`bufferEncoding`** - how to encode binary `Buffer` contents into a string. The value gets passed to Node's `Buffer#toString()`. Default: `base64`

Notes on how values are converted:

* Strings, booleans, numbers and nulls are copied as is
* Arrays and objects are processed recursively
* Dates are stringified using `#toString()`
* Buffers are stringified using `#toString()` with encoding specified in options

<a id="serializer-default-options"></a>
#### `.defaultOptions`

Object holding the global defaults for serialization [options](#options).

<a id="schemes"></a>
### Defining schemes

#### Scheme object

Scheme are defined as plain objects. They can contain following fields:

* **`include`** - a list of attributes to be included in resulting objects. Defaults to all attributes (`['@all']`). See [Attribute lists](#attribute-lists) for details.
* **`exclude`** - a list of attributes to be excluded from the result. Filtering is applied to the attribute list defined by `include`. Defaults to an empty list. See [Attribute lists](#attribute-lists) for details.
* **`assoc`** - an object containing schemes for associated model instances. Object's keys should correspond to names of attributes holding related instances (Sequelize's `as`). Values can be either scheme names or scheme objects. They will be passed to the `Serializer` constructor when creating serializers for associated instances.
* **`as`** - can be used to rename attributes in output. Should be an object mapping model attribute names to names we would like to have in JSON. Useful for naming properties obtained from method calls (eg. to have `postExcerpt` instead of `getPostExcerpt`).
* **`options`** - serializer [options](#options). They will override options passed to the `Serializer` constructor.
* **`postSerialize`** - the hook function to be called after the instance has been converted into object. It receives the output object as the first argument and the original instance as the second one. Must return the (modified) output object. Gets called *after* the [model-wide `postSerialize` hook](#schemes-inside-models).

#### Attribute lists

Both the `include` and the `exclude` fields of a scheme definition should contain a list of model's attributes. But, these lists can be a little more than just plain arrays of attribute names. They can also contain:

* method names (to be called with no arguments),
* regular instance properties,
* `@`-prefixed selectors that expand to subsets of model attributes sharing certain features.

Attributes can be prefixed with a dot to mark them expicitely as regular attributes of the model instance object.

`sequelize-to-json` provides support for following attribute selectors:

* **`@all`** - all attributes defined for the model
* **`@assoc`** - all associations in the model
* **`@pk`** - the primary key (if present)
* **`@fk`** - all foreign keys
* **`@doc`** - all document-type fields: `JSON`, `JSONB`, `HSTORE`
* **`@blob`** - all `BLOB`s
* **`@virtual`** - all `VIRTUAL` attributes
* **`@auto`** - all attributes auto-generated by Sequelize (eg. `created_at`, `updated_at`...)

Note: model attribute values are obtained by calling `.get()` on the model instance. This means custom getters get executed.

<a id="schemes-inside-models"></a>
#### Schemes inside models

In order to keep things clean, serialization schemes can be kept inside models. This is done by adding a static `serializer` property to the model class. You can either set this property directly or just put it in `classMethods` when defining the model.

Supported `serializer` fields are:

* **`schemes`** - an object containing available serialization schemes (keys are names and values are scheme objects)
* **`defaultScheme`** - name of the default scheme for this model. The name should exist as a proper key in `schemes`.
* **`options`** - model-wide defaults for [serialization options](#options)
* **`postSerialize`** - the model-wide hook function to be called after the instance has been converted into object. It gets called with the serialization scheme as `this` and receives 3 arguments: the output object, the original model instance and the name of the serialization scheme. Must return the (modified) output object. Gets called *before* the scheme-specific `postSerialize` hook.

### Options

Following options can be used to tune the serialization output:

* **`encoder`** - a function used to convert JS types to JSON-friendly format. It should accept an object to be encoded and can be also passed options. Default: [`Serializer.encodeToJSON()`](#serializer-encode-to-json)
* **`undefinedPolicy`** - what to do with attributes that are `undefined` for the instance. Allowed policies are `Serializer.SKIP` (exclude from output), `Serializer.SET_NULL` (set their values to `null`) and `Serializer.FAIL` (throw an error). Default: `Serializer.SKIP`
* **`copyJSONFields`** - whether values stored as JSON (`JSON`, `JSONB` etc.) should be copied directly without passing them through `encoder`. Having it on will improve performance a bit. Default: `true`
* **`simpleDates`** - whether `DATEONLY` fields should be encoded in `YYYY-MM-DD` format with the timezone offset applied. If set to `false`, they will be stringified as full date-times. Default: `true`. Note: In Sequelize 4 `DATEONLY` fields are implemented as strings model-side so this option will have no effect.
* **`encoderOptions`** - options to be passed to `encoder` (as the second argument)
* **`attrFilter`** - function to filter model attributes before serialization. Can be useful in certain cases, eg. when one wants to get rid of duplicates of auto-generated attributes (such as `some_id` showing up together with `someId`). The function gets passed 2 arguments: the attribute object and the model object. It should return `false` for the attribute to be excluded from serialization.

Above options can be customized in four different places altogether (with precedence from the top to the bottom):

* in [constructor](#serializer-serializer) (passed as the last argument);
* inside a [scheme object](#schemes);
* [inside a model](#schemes-inside-models);
* globally, by modifying `Serializer.defaultOptions`.

## Tips

### Adding serialization methods to models

To save yourself some typing and `require`ing across the application code, you can add serialization methods globally to all your models. Just use `define.classMethods` and `define.instanceMethods` options during Sequelize instantiation:

```js
const db = new Sequelize(..., {
  //...
  define: {
    classMethods: {
      serializeMany: function(data, scheme, options) {
        return Serializer.serializeMany(data, this, scheme, options);
      },
      //...
    },
    instanceMethods: {
      serialize: function(scheme, options) {
        return (new Serializer(this.Model, scheme, options)).serialize(this);
      },
      //...
    }
  },
  //...
});

```

Now you can use these methods like so:

```js
// for single object
let json = myModelInstance.serialize('someScheme');
// for many objects
let json = MyModel.serializeMany(instances, 'someScheme');
```

Of course you can provide the scheme as an object and pass serializer options as the last argument for both methods.


