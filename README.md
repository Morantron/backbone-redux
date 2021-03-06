backbone-redux
=========================

The easy way to keep your backbone collections and redux store in sync.

[![npm](https://img.shields.io/npm/v/backbone-redux.svg?style=flat-square)](https://www.npmjs.com/package/backbone-redux)
[![npm](https://img.shields.io/npm/dm/backbone-redux.svg?style=flat-square)](https://www.npmjs.com/package/backbone-redux)
[![Travis](https://img.shields.io/travis/redbooth/backbone-redux.svg?style=flat-square)](https://travis-ci.org/redbooth/backbone-redux)

```
npm install backbone-redux --save
```

Creates reducers and listeners for your backbone collections and fires action creators on every collection change.

**Documentation is a work-in-progress**. Feedback is welcome and encouraged.

### Why?

* You can start migrating your apps from backbone to react+redux in no time.
* No need to worry about migrated/legacy parts of your app being out of sync, because both are using the single source of truth.
* No boilerplate.
* You can hide all new concepts like `reducers`, `stores`, `action creators`, `actions` and `purity` from other developers in your team to avoid brain overloading.
* You have REST-adapter to your server out-of-the-box. Most React projects end up implementing an ad hoc, bug-ridden implementation of Backbone.Collection not only once, but for each store.
* You have separation between server-data and UI-data. The later is flat, so working with it is a plesure in React.

### How to use?
#### Auto way


```javascript
import { createStore, compose } from 'redux';
import { devTools } from 'redux-devtools';
import { syncCollections } from 'backbone-redux';

//  Create your redux-store, include all middlewares you want.
const finalCreateStore = compose(devTools())(createStore);
const store = finalCreateStore(() => {}); // Store with just empty dumb reducer

// Now just call auto-syncer from backbone-redux
// Assuming you have Todos Backbone collection globally available
syncCollections({todos: Todos}, store);
```

What will happen?

* `syncCollections` will create a reducer under the hood especially for your collection.
* `action creator` will be constructed with 4 possible actions: `add`, `merge`, `remove`, and `reset`.
* Special `ear` object will be set up to listen to all collection events and trigger right actions depending on the event type.
* Reducer will be registered in the store under `todos` key.
* All previous reducers in your store will be replaced.  

You are done. Now any change to `Todos` collection  will be reflected in the redux store.

Models will be serialized before saving into the redux-tree: a result of calling `toJSON` on the model + field called `__optimistic_id` which is equal to model's `cid`;

Resulting tree will look like this:

```javascript
{
  todos: {
    entities: [{id: 1, ...}, {id: 2, ...}],
    by_id: {
      1: {id: 1, ...},
      2: {id: 2, ...}
    }
  }
}
```

`entities` array is just an array of serialized models. `by_id` — default index which is created for you. It simplifies object retrieval, ie:
`store.getState().todos.by_id[2]`

So, what is happening when you change `Todos`?

```
something (your legacy/new UI or anything really) changes Todos
  -> Todos collection emits an event
    -> ear catches it 
      -> ActionCreator emits an action
        -> Reducer creates a new state based on this action
          -> New State is stored and listeners are notified
            -> React doing its magic
```

##### Configuration options

**Index maps**

You can remove/change the indexes that will be created for your collections. To do that, you need to pass `indexes_map` param in your collections object.

For example, I have a `people` collection of models with 4 fields: `name`, `id`, `token`, and `org_id`. And I want to have indexes for all fields except `name`.

```javascript
const jane = new Backbone.Model({id: 1, name: 'Jane', org_id: 1, token: '001'});
const mark = new Backbone.Model({id: 2, name: 'Mark', org_id: 2, token: '002'});
const sophy = new Backbone.Model({id: 3, name: 'Sophy', org_id: 1, token: '003'});
const people = new Backbone.Collection([jane, mark, sophy]);

const indexesMap = {
  fields: {
    by_id: 'id',
    by_token: 'token'
  },
  relations: {
    by_org_id: 'org_id'
  }
};

syncCollections({
  people: {
    collection: people,
    indexes_map: indexesMap
  }
}, store);

/**
  store.getState().people =>

  { 
    entities: [
      {id: 1, name: 'Jane', org_id: 1, token: '001', __optimistic_id: 'c01'},
      {id: 2, name: 'Mark', org_id: 2, token: '002', __optimistic_id: 'c02'},
      {id: 3, name: 'Sophy', org_id: 1, token: '003', __optimistic_id: 'c03'}
    ],
    by_id: {
      1: {id: 1, name: 'Jane', org_id: 1, token: '001', __optimistic_id: 'c01'},
      2: {id: 2, name: 'Mark', org_id: 2, token: '002', __optimistic_id: 'c02'},
      3: {id: 3, name: 'Sophy', org_id: 1, token: '003', __optimistic_id: 'c03'}
    },
    by_token: {
      '001': {id: 1, name: 'Jane', org_id: 1, token: '001', __optimistic_id: 'c01'},
      '002': {id: 2, name: 'Mark', org_id: 2, token: '002', __optimistic_id: 'c02'},
      '003': {id: 3, name: 'Sophy', org_id: 1, token: '003', __optimistic_id: 'c03'}
    },
    by_org_id: {
      1: [
        {id: 1, name: 'Jane', org_id: 1, token: '001', __optimistic_id: 'c01'},
        {id: 3, name: 'Sophy', org_id: 1, token: '003', __optimistic_id: 'c03'}
      ],
      2: [
        {id: 2, name: 'Mark', org_id: 2, token: '002', __optimistic_id: 'c02'}
      ]
    }
  }
  */
```

And to remove indexes at all, just pass empty object as `indexes_map` for `syncCollections`.

**Serializers**

If you want something different as a result of serialization to be stored in the tree, just pass a function as a `serializer` key to `syncCollections`:

```javascript
const jane = new Backbone.Model({id: 1, name: 'Jane'});
const collection = new Backbone.Collection([jane]);

const serializer = (model) => ({id: model.id, name: `${model.get('name')} Janes`, __optimistic_id: model.cid });

syncCollections({
  people: {
    collection,
    serializer
  }
}, store);

/**
  store.getState().people =>

  { 
    entities: [
      {id: 1, name: 'Jane Janes', __optimistic_id: 'c01'},
    ],
    by_id: {
      1: {id: 1, name: 'Jane Janes', __optimistic_id: 'c01'},,
    }
  }
  */
```

**Extra reducers**

`syncCollection` by default will replace all reducers in your store with new generated ones. To add some non-generated reducers into the store, you can pass them as a third param:

```javascript
const extraReducer = (state = {}) => state;
syncCollections({people: collection}, store, {some_extra_branch: extraReducer});
```

#### Manual Artesanal Way

Sometimes defaults that are provided by `syncCollections` are not enough. 

Reasons could vary:
* your collection could not be globally available
* you need some custom rules when adding/removing/resetting collection
* your collection have any dependency that should be processed too
* etc

In all these cases you can't use `syncCollections`, but you can create your own ears to mimic `syncCollections` behaviour.

Any `ear` should look something like this:

```javascript
import { bindActionCreators } from 'redux';

export default function(collection, rawActions, dispatch) {
  // binding action creators to the dispath function
  const actions = bindActionCreators(rawActions, dispatch);

  actions.add(collection.models); // initial sync
  
  // adding listeners
  collection.on('add', actions.add);
  collection.on('change', actions.merge);
  collection.on('remove', actions.remove);
  collection.on('reset', ({models}) => actions.reset(models));
}
```

As you can see, `ear` requires 3 attributes. `collection` and `dispatch`(this is just `store.dispatch`) you normally should already have, but how we can genarate `rawActions`? You can use `actionFabric` that `backbone-redux` provides:

```javascript
import {actionFabric} from 'backbone-redux';

// create some constants that will be used as action types
const constants = {
  ADD: 'ADD_MY_MODEL',
  REMOVE: 'REMOVE_MY_MODEL',
  MERGE: 'MERGE_MY_MODEL',
  RESET: 'RESET_MY_MODEL'
};

// you need some serializer to prepare models to be stored in the store.
// This is the default one that is used in backbone-redux,
// but you can create totally your own, just don't forget about __optimistic_id
const defaultSerializer = model => ({...model.toJSON(), __optimistic_id: model.cid});

export default actionFabric(constants, defaultSerializer);
```

Don't forget that `actionFabric` is just an object with a couple of methods, you can extend it as you want.

Time to generate a reducer:

```javascript
import {reducerFabric} from 'backbone-redux';

// the same constants, this is important
const constants = {
  ADD: 'ADD_MY_MODEL',
  REMOVE: 'REMOVE_MY_MODEL',
  MERGE: 'MERGE_MY_MODEL',
  RESET: 'RESET_MY_MODEL'
};

// any indexes that you want to be created for you
const index_map = {
  fields: {
    by_id: 'id'
  }
, relations: {
    by_channel_id: 'channel_id'
  }
};

export default reducerFabric(constants, index_map);
```


And now we are ready to combine everything together:

```javascript
import { syncCollections } from 'backbone-redux';
import store from './redux-store';
import customReducer from './reducer';
import customEar from './ear';
import customActions from './actions';

export default function() {
  // start with syncing normal collections
  const collectionsMap = {
    collection_that_does_not_need_customization: someCollection
  };
  
  // we need to pass our prepared reducers into the store
  // if you don't use syncCollections at all, you just need 
  // to create store normally with these reducers via
  // combineReducers from redux
  const extraReducers = {
    custom_collection: customReducer
  };

  syncCollections(collectionsMap, store, extraReducers);

  // now let's call the ear
  customEar(customCollection, customActions, store.dispatch);
}
```

Done, you have your custom ear placed and working.


### Examples

* [TodoMVC](https://github.com/redbooth/backbone-redux/tree/master/examples/todomvc)

### Licence
MIT
