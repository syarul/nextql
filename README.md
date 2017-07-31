# ![nextql logo](images/nextql.png) NextQL 

NextQL is JSON query language for APIs and a extremely flexible runtime for resolve those queries.
1. Ask what you need, get exactly that. 
2. Get many resource by a single request.
3. No limitation how to define type system.
4. No limitation how to resolve request

## Introduction to NextQL
Instead of complex type system like GraphQL, NextQL use plain object to describe how fulfill data queries.

For example a User model:
```js
{
    fields: {
        id: 1, /* 1 mean NextQL should automatically resolve the field type */
        firstName: 1,
        lastName: 1
    },
    computed: {
        fullName(user){
            return user.firstName + ‘ ’ + user.lastName;
        }
    },
    methods:{
        me(params, ctx){
            return ctx.request.auth.user;
        }   
    }
}
```
User model tell NextQL - user object may have 3 fields id, firstName and lastName. The user also have a computed field fullName which calculated use the function provided. Finally the user model expose a me function to query. Compare with GraphQL, NextQL distinguish between a normal field, a computed field and a query function. So you don’t need to group all expose queries into single root Query type like GraphQL - remember many people ask [Can I split queries and mutators across different files?](https://github.com/apollographql/graphql-tools/issues/186).

When NextQL receive NextQL query which in JSON format, it will start to resolve follow order: model -> method -> fields -> ... recursive fields -> final result.

For example the query
```json
{
    "user": {
        "me": {
            "fullName": 1
        }
    }
}
```

Could produce the JSON result:
```json
{
    "user":{
        "me": {
            "fullName": "Giap Nguyen Huu"
        }
    }
}
```

## Query
NextQL query is a JSON object define what API methods called and what data to return. 

```json
{
    "user": {  
        "me": { 
            "fullName": 1
        }

    }
}
```

Equivalent call **me** method of class **user** then pick **fullName** field from result. It look like combine REST API call with GraphQL query. 
```
/user/me => { fullName }
```

### Arguments
NextQL allow pass arguments to methods and computed fields via reserved **$params** field.

```json
{   
    "human": {
        "get": {
            "$params": { "id": "1000" },
            "fullName": 1,
            "height": {
                "$params": { "unit": "m"}
            }
        }
    }
}
```

Could produce the JSON result:
```json
{
    "human":{
        "get": {
            "$params": { "id": "1000" },
            "fullName": "Nguyen Huu Giap",
            "height" : 1.69
        }
    }
}
```

### Alias
Because result field match with query field. If you need call multiple methods, fields you need alias. NextQL alias is a suffix separator which resolver ignore.
```json
{
    "human":{
        "get/1000": {
            "$params": { "id": "1000" },
            "name": 1
        },
        "get/1001": {
            "$params": { "id": "1001" },
            "name": 1
        }
    }
}
```

Could produce the JSON result:
```json
{
    "human":{
        "get/1000": {
            "name": "Nguyen Huu Giap"
        },
        "get/1001": {
            "name": "Dinh Thi Kim Nguyen"
        }
    }
}
```

By default **/** is alias separator, anything after it doesn't counted. You could config any suffix separator.

### Traverse related object
You can ask more data from relate objects. 

```json
{
	"person": {
		"get/giapnh": {
			"$params": { "id": "giapnh" },
			"name": 1,
			"children": {
				"name": 1
			}
		},
		"get/nguyen": {
			"$params": { "id": "nguyen" },
			"name": 1,
			"children": {
				"name": 1
			}
		}
	}
}
```

The JSON result should be
```json
{
	"person": {
		"get/giapnh": {
			"name": "Nguyen Huu Giap",
			"children": [{
				"name": "Nguyen Huu Vinh"
			}]
		},
		"get/nguyen": {
			"name": "Dinh Thi Kim Nguyen",
			"children": [{
				"name": "Nguyen Huu Vinh"
			}]
		}
	}
}
```

## Schema
NextQL schema is a groups of models defined as plain Javascript object
```js
const personModel = {
	fields: {
		firstName: 1, /* let NextQL decide field type */
		lastName: 1,
        address: "address" // explicit field type
        phone: { /* define inline type */
            work: 1,
            home: 1
        }
	},

	computed: {
		fullName(source, params, context){
			return source.firstName + source.lastName;
		}
	},

	methods: {
		get(params, context){
			return context.db.get(params.id);
		}
	}
}
```

* **fields**: Any field exposed for query. Resolve directly from source.
* **computed**: Apply for virtual field which not present in source object or you want expose source object's methods.
* **methods**: Any methods the model expose for query.

There are 3 options to define field type describe above
1. Let NextQL resolve field type. This is prefer option.
2. Explicit assign model name.
3. Inline nested fields.

computed fields and methods automatically let NextQL resolve return type.

### How NextQL decide field type?
NextQL use a global **resolveType function** to resolve model name from object which by default use value constructor name for model name. 
```js
const defaultResolveType = value => value.constructor && value.constructor.name;
```

You can config your own **resolveType** or better use **afterResolveTypeHooks**. You free to choose whatever to resolve type from object. It could be mongoose model name, __type field ... 
If resolveType return "Object" as model name, NextQL will fall back to default behavior: It only allow to query primitive field value. This behavior help you free from define unnecessary simple models.   

## NextQL :heart: plugins
NextQL very simple and flexible. Everything could extensible/customize. NextQL follow Vue plugin pattern.

```js
MyPlugin.install = function (nextql, options) {
  nextql.beforeCreate(schema => schema);
  nextql.afterResolveType(source => source.__type);
}

nextql.use(MyPlugin);
```

* **nextql.beforeCreate** : the hook call before NextQL build Model object from schema. It is powerful hook to customize schema.
* **nextql.afterResolveType** : the hook call after NextQL resolve type from source object. It give you a chance to map source to NextQL model type.

Sample Mongoose plugin - it catch any schema have mongoose option:
* Create mongoose model from schema fields.
* Inject CRUD methods into schema methods.

Finally it help resolve mongoose document into NextQL model.

```js
const mongoose = require('mongoose');

function hookBeforeCreate(options) {
	if (options.mongoose) {
		const model = mongoose.model(options.name, options.fields);
		options.methods = Object.assign({
			get({ id }) {
				return model.findById(id);
			},
			find() {
				return model.find()
			},
			create({ data }) {
				var ins = new model(data);
				return ins.save();
			},
			update({ id, data }) {
				return model.findById(id).then(
					ins => {
						Object.keys(data).forEach(path => ins.set(path, data[path]));
						return ins.save();
					}
				)
			},
			remove({ id }) {
				return model.findByIdAndRemove(id);
			}

		}, options.methods);
	}
}

function hookAfterResolveType(source) {
	return source.constructor && source.constructor.modelName;
}

module.exports = {
	install(nextql) {
		nextql.beforeCreate(hookBeforeCreate);
		nextql.afterResolveType(hookAfterResolveType);
	}
}
```

**Mongoose plugin in action**
```js
const mongoose = require("mongoose");
mongoose.Promise = global.Promise;

mongoose.connect('mongodb://localhost/nextql', { useMongoClient: true }).catch(error => {
	console.log(error);
	process.exit(1);
});

const nextql = require('../nextql');
const nextqlMongoose = require('./index');
nextql.use(nextqlMongoose);

nextql.model('test', {
	mongoose: true,
	fields: {
		_id: String,
		name: String
	}
});

async function run() {
	const users = await nextql.execute({
		test: {
			find: {
				name: 1
			}
		}
	});

	return users.test
}
```

Combine beforeCreate hook and afterResolveType hook, you able to create any kind of NextQL schema and behaviors.

## Installing / Getting started


## Developing


### Building



### Deploying / Publishing


## Features

## Licensing

"The code in this project is licensed under MIT license."