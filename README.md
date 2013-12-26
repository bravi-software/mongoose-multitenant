Mongoose Multitenant
=====================

This package facilitates horizontal multitenancy in one MongoDB database, obviously using Mongoose as the interaction layer. 

## Basics
With this package, you can use one schema per model, as you would normally with Mongoose, and then use special methods and syntax apply that schema to different tenant collections. 

Instead of using `mongoose.model(name, schema)` to compile your model, you would now use `mongoose.mtModel(name, schema)`. This still creates the Mongoose model as normal, but adds some additional functionality. Specifically, you can retrieve a model for a specific tenant using this syntax:

`mongoose.model('tenantId.modelName')`

When that happens, the package will check if that model for that tenant has already been compiled. If not, it creates a copy of the base model's schema, updates any `refs` to other collections, and then compiles a new model with the new schema, with a collection name of `tenantId__originalCollectionName`. All per-tenant models are *lazy-loaded*, meaning that they won't take up memory until they are needed.

## Usage
#### Pull in requirements
```javascript
var mongoose = require('mongoose');
require('mongoose-multitenant');

mongoose.connect('mongodb://localhost/multitenant');
```

#### Create a schema
With mongoose-multitenant you use all the same syntax for schema creation as you normally do with Mongoose, with the addition of the `$tenant` property on document references. This tells the system whether it is a reference to a document for the same tenant (`true`) or a root-level document without a tenant (`false`)

```javascript
var barSchema = new mongoose.Schema({
	title:String,
	_foos:[{
		type:mongoose.Schema.Types.ObjectId,
		ref:'Foo',
		$tenant:true
	}]
});

var fooSchema = new mongoose.Schema({
	title:String,
	date:Date
});
```

#### Compile the models
Instead of using `mongoose.model` to compile, you use `mongoose.mtModel`.

```javascript
mongoose.mtModel('Bar', barSchema);
mongoose.mtModel('Foo', fooSchema);
```

#### Use the models
Basic usage:
```javascript
// This returns a new Foo model for tenant "tenant1"
var fooConstructor = mongoose.mtModel('tenant1.Foo');
var myFoo = new fooConstructor({
	title:'My Foo',
	date:new Date()
});

myFoo.save(function(err, result) {
	// This saved it to the collection named "tenant1__foos"
});
```

And make use of refs/populate:
```javascript
var barConstructor = mongoose.mtModel('tenant1.Bar');
var myBar = new barConstructor({
	title:'My Bar'
	_foos:[myFoo._id]
});

myBar.save(function(err, result) {
	// Saved to the collection named "tenant1__bars"

	barConstructor.find().populate('foos').exec(function(err, results) {
		console.log(results[0]._foos[0].title); // "My Foo"
	});
});
```

But you can't populate across tenancies:
```javascript
var tenant2Bar = mongoose.mtModel('tenant2.Bar');
var newBar = new tenant2Bar({
	title:'New Bar',
	_foos:[myFoo._id]
});

newBar.save(function(err, result) {
	tenant2Bar.find().populate('foos').exec(function(err, results) {
		console.log(results[0]._foos[0]); // "undefined"
	});
});
```

## Credits
* Brian Kirchoff for his great mongoose-schema-extend package