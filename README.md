COMPUTED
================
[![Greenkeeper badge](https://badges.greenkeeper.io/fullcube/loopback-ds-computed-mixin.svg)](https://greenkeeper.io/)

[![CircleCI](https://circleci.com/gh/fullcube/loopback-ds-computed-mixin.svg?style=svg)](https://circleci.com/gh/fullcube/loopback-ds-computed-mixin) [![Coverage Status](https://coveralls.io/repos/github/fullcube/loopback-ds-computed-mixin/badge.svg?branch=master)](https://coveralls.io/github/fullcube/loopback-ds-computed-mixin?branch=master) [![Dependencies](http://img.shields.io/david/fullcube/loopback-ds-computed-mixin.svg?style=flat)](https://david-dm.org/fullcube/loopback-ds-computed-mixin) [![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)

This is a mixin for the LoopBack framework that adds computed properties to a model.

A computed property is a property of which the value is set dynamically after reading model data from the data source.



> Forked [loopback-ds-computed-mixin](https://github.com/fullcube/loopback-ds-computed-mixin) for self use by adding  fields query support to computed properties which is realized in its pull request but not been merged.
> This is useful to avoid **cycle reference** from two models. 

Suppose in Model named Course we have a computed property called courseDiscount which calls method **find** and triggers observer **loaded**
```javascript 
  Course.courseDiscount = async instance => {
    var CourseDiscount = Course.app.models.CourseDiscount
    var coursediscounts = await CourseDiscount.find()
    var id = instance.id
    var courseDiscount = coursediscounts.find(item => {
      if (!item.courseIds) return false
      return item.courseIds.indexOf(id) > -1
    })
    console.log(courseDiscount)
    return courseDiscount || { discount: 0 }
  }
```

In the Corresponding Model CourseDiscount the observer **loaded** of Course is triggered and the cycle reference will cause no responding.
```javascript 
  var courseDiscount
  Coursediscount.courses = async instance => {
    var Course = Coursediscount.app.models.Course
    if (instance && instance.courseIds) {
      var ids = instance.courseIds
      /* raw data loaded from database is json, which must be parsed */
      if (typeof ids === 'string')
        ids = JSON.parse(ids)
      /* remove the computed in Course to avoid circle reference
         courseDiscount must not be local variable and must be set when Course.courseDiscount!==undefined
         because it's shared by multi threads, which may cause Course.courseDiscount can't be recovered forever.
       */
      if (Course.courseDiscount) {
        courseDiscount = Course.courseDiscount
      }
      /* can't use delete ,which indeed remove the element forever */
      Course.courseDiscount = undefined
      var data = await Course.find({ where: { id: { inq: ids } }, include: 'product' })
      /* it's very possible Course.courseDiscount is recovered when another thread is getting data and finally stucked
         so it's wise to recover it the next tick
       */
      process.nextTick(() => {
        Course.courseDiscount = courseDiscount
      })
      return data
    }
  }
```
Now it's only needed to add a fields query
```javascript
await Course.find({ where: { id: { inq: ids } }, include: 'product',fields:{courseDiscount:false} })
```
- The mixin uses the `loaded` observer.
- It only runs when a single instance gets loaded, e.g. it checks `ctx.instance`.
- It only runs when it is a new instance, e.g. it checks `ctx.isNewInstance`.
- It overrides the configured property if one exists in the data source.

INSTALL
=============

```bash
npm install --save loopback-v-computed-mixin
```

SERVER CONFIG
=============
Add the mixins property to your server/model-config.json:

```
{
  "_meta": {
    "sources": [
      "loopback/common/models",
      "loopback/server/models",
      "../common/models",
      "./models"
    ],
    "mixins": [
      "loopback/common/mixins",
      "../node_modules/loopback-v-computed-mixin/lib",
      "../common/mixins"
    ]
  }
}
```

CONFIG
=============

To use with your Models add the `mixins` attribute to the definition object of your model config.

The property you want to compute has to be defined in the model. The callback can be a promise too.

```json
{
    "name": "Item",
    "properties": {
        "name": "String",
        "description": "String",
        "status": "String",
        "readonly": "boolean"
    },
    "mixins": {
        "Computed": {
            "properties": {
                "readonly": "computeReadonly"
            }
        }
    }
}
```

On your model you have to define the callback method.

```javascript
// Set an item to readonly if status is archived
Item.computeReadonly = function computeReadonly(item) {
  return item.status === 'archived';
};

```

TESTING
=============

Run the tests in `test.js`

```bash
  npm test
```

Run with debugging output on:

```bash
  DEBUG='loopback:mixin:computed' npm test
```
