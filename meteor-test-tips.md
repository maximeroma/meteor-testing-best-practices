## Unit Testing in Meteor 1.3+

The key to successful application testing is *reliability* and in my experience, the larger an application grows, the less reliable a test suite becomes. There can be several reasons for this:

1. race conditions - interference within test suite
1. cascading failure - when one test fails everything after it fails
1. non standard source code - the same result is achieved many different ways in the application
1. non standard test code - code becomes too complex as many developers write more tests

#### Objectives

By the end of this tutorial you will be able to:
* set up unit tests geared toward a large project within your Meteor application
* be able to define a standard testing structure for your team that also enforces standards in your source code
* use promises to make a more consistent, repeatable testing environment


### Testing Meteor: client or server, pick one

Meteor is an isomorphic framework, meaning that the same code can be run on either the server or the client ... or ideally both. When running Meteor in production this allows us to take advantage of one of Meteor's key offerings, the optimistic UI, or eventual consistency with our DB. However, when running tests this isomorphic structure can tangle up our process. Can you guess how?

Consider the following code:

```javascript

Meteor.methods({
  isLoggedIn: => {
    return Meteor.user() ? true : false;
  }
})

```

The test:

```javascript
describe( 'the isLoggedIn function', => {
  it('should return false if a user is NOT logged in', (done) => {
    expect(Meteor.call('isLoggedIn')).to.be.false;
    done()
  })
  it('should return true if a user is logged in', (done) => {
    sinon.stub(Meteor, 'user', => true)
    expect(Meteor.call('isLoggedIn')).to.be.true;
    Meteor.user.retore()
    done()
  })
})

```

Very straight forward. So why does it behave as expected sometimes and not others? Whats more, there seems to be a pattern, every other time the tests pass on the client, then on the server...infuriating if you have ever been here.

Gotcha! The STUBS! They are being stubbed and then restored on both the client and the server ... _simultaneously_ .

The solution is to run your method tests only on the server. This is sufficient as the server is your source of truth. If the tests pass on the server, then we know the client is either in sync or will be shortly after Meteor's optimistic UI magic has done its work.

```javascript
if (Meteor.isServer) {

  describe( 'the isLoggedIn function', => {
    it('should return false if a user is NOT logged in', (done) => {
      expect(Meteor.call('isLoggedIn')).to.be.false;
      done()
    })
    it('should return true if a user is logged in', (done) => {
      sinon.stub(Meteor, 'user', => true)
      expect(Meteor.call('isLoggedIn')).to.be.true;
      Meteor.user.retore()
      done()
    })
  })

}

```

Consistency is the name of the game in reliable unit tests.

So then what unit tests would you run on the client? Anything that is not looking at the database can/should be run on the client such as a UI helper, or processing user input.


### Advanced Structure for Testing in Meteor

Here we are going to define an advanced structure for your tests to keep things explicit, efficient and scaleable. The keys to this structure will be:

* taking advantage of the *imports* folder and the *test* folder
* using a test runner file for ease of isolation and synchronicity

In a shift towards making Meteor more inline with Javascipt trends and best practices, Meteor 1.3 introduced lazy loading, which allows developers to only load files/assets that are needed for a specific feature. This isolation can be leveraged primarily for speed, simplicity, organization, and security. In making this shift, it seems that backward compatibility was a major concern for the Meteor Core developers, so this lazy loading is _opt in only_. Amazing for all the people with existing Meteor apps that want to take advantage of other great additions that come along in the 1.3 release, native testing being one of them. This opt in approach is achieved through a name spaced directory in the root of the project called *imports*. Anything in this folder will only be loaded when a file explicitly calls for it through an `import` or `require` statement.

> We won't visit it in this tutorial but it is worth mentioning that a *test* directory has also been name spaced and will only load when Meteor is running in a test environment. This is huge bonus for security reason as we can put some destructive methods in there that are essential for quality testing, but could wreak havoc if they fell into the wrong hands in production.

Using the *imports* directory, lets build a tiered testing structure!

Our app structure will look something like this:

root
  |- client
  |- imports
    |- test
        |- methods
  |- methods
    |- user.js
    |- item.js
  |- server
  |- test
    |- utils
  |- testRunner.js


*methods/user.js*
```javascript

Meteor.methods({
  createUser: (...) => {
    ...
  },

  getUser: (...) => {
    ...
  },

  updateUser: (...) => {
    ...
  },

  removeUser: (...) => {
    ...
  }

})

```
*methods/item.js*
```javascript

Meteor.methods({
  createItem: (...) => {
    ...
  },

  getItem: (...) => {
    ...
  },

  updateItem: (...) => {
    ...
  },

  removeItem: (...) => {
    ...
  }

})

```

#### testRunner.js

Lets breakdown this structure starting from the outside and moving inwards.

``` javascript
import user from 'imports/test/methods/user.js'
import item from 'imports/test/methods/item.js'

let testRunner = Promise.resolve();

testRunner
  .then(=> user.tests())
  .then(=> item.tests())
  .catch((err) => console.log('There was an error in the test runner', err))
```

Here we have established the file that will be called to run our tests. The promises are a convenient way to organize the code and to create an extra enforcement of synchronicity. This structure is a solution to false positives/negative because of race conditions within our testing suite. Another benefit is the ability to isolate a set of test for rapid feedback and iteration in  test driven development.

Lets have a look at the next layer in.

*imports/test/methods*

As you can see, we have two sets of methods: user and item. There are basic CRUD (Create Read Update Delete) methods in both item.js and user.js. That means there are 4 methods per file. Lets make a file inside of imports/test/ to test each of these methods like so:

imports
  |- test
    |- user
      |- user.js
      |- createUser.js
      |- getUser.js
      |- updateUser.js
      |- removeUser.js
    |- item
      |- item.js
      |- createItem.js
      |- getItem.js
      |- updateItem.js
      |- removeItem.js


Whats up with the additional files, _user.js_ and _item.js_? These will be the 'containers' for each set of method tests, much like the testRunner.js is the container for our whole test suite. A container in this case groups all of our tests in this directory and will be imported later into our master testRunner file. This accomplishes a super clear structure that allows a developer to go straight to a file when a test fails to see whats going on under the hood. As well when we add a new method test, we simply make a file for it and add it to the container for that directory.


*imports/test/user/user.js*

```javascript
import createUser from './createUser.js'
import getUser from './getUser.js'
import updateUser from './updateUser.js'
import removeUser from './removeUser.js'

let tests = => {
  if (Meteor.isServer) {
    describe('user methods', => {
      describe('createUser', createUser)
      describe('getUser', getUser)
      describe('updateUser', updateUser)
      describe('removeUser', removeUser)
    })
  }
}

export user = {
  tests
}

```
As you can see, we are creating a function called `tests` that we are exporting from this file. The 'tests' function calls each one of our method tests.

*imports/test/user/createUser.js*

```javascript
createUser = =>
  describe('throws errors', => {
    it('throws an error if user is not logged in', (done) => {
      ...
    })
    it('throws an error if ...', (done) => {
      ...
    })
  })
  describe('success', => {
    it('successfully creates a new user', (done) => {
      ...
    })
    it('it sets the users information correctly', (done) => {
      ...
    })
  })

export createUser
```

Here we actually define our `it` statements and testing for proper behavior, both failure and success. This pattern allows for a clear read out in the browser view when the tests are run.


### Test Utils and Promises

Creating a set of test utilities gives structure to your code before you even write it! As well, using promises as the default for testing our Meteor methods makes the code explicitly synchronous and easy to read/refactor.

Consider this code:

```javascript

Meteor.methods({

  updateDocument: (docId, collectionName, updates) => {
    check(docId, String)
    check(collectionName, String)
    check(updates, Array)

    if (!Meteor.user()) {
      throw new Meteor.error('not-logged-in', 'You must be logged in to access this function!')
    }

    if (!Roles.userIsInRole(Meteor.userId(), 'manageDocs')) {
      throw new Meteor.error('not-permitted', `You do not have permission to access this function.`)
    }

    doc = db[collectionName].findOne({_id: docId})

    doc.set(updates)

    if (!doc.validate()) {
      throw new Meteor.error('doc-invalid', `This is not a valid update.`)
    }

    doc.save()

  }

})

```

If this is how we want our code to be structured, then we can expect to be writing all methods in a similar pattern:

1. Check the arguments
1. Make sure a user is logged in
1. Make sure user has permission
1. Make sure a document exists with that id
1. Make sure the updated document is valid


```javascript
if (Meteor.isServer) {
  describe( 'updateDocument function', => {

    it('should throw if the first argument is not a string', (done) => {
      ...
    });

    it('should throw if the second argument is not a string', (done) => {
      ...
    });

    it('should throw if the third argument is not an object', (done) => {
      ...
    });

    it('should throw if a user is not logged in', (done) => {
      ...
    });

    it('should throw if a user does not have permission', (done) => {
      ...
    });

    it('should throw if no document is found with passed id', (done) => {
      ...
    })

    it('should throw if validation was not run', (done) => {
      ...
    })

    // ... finally on to the success case
  })

}

```

Thats a lot of tests to check the standard function and security of an argument, but super important if you ask me. Here are few utils that you can use to make this process more standardized as well as encourage best practices within your team. These tests assume you are using a couple of great packages built specifically for Meteor:
  * *check*, used for ensuring arguments are what we expect them to be
  * Alanning Roles, which provides a robust permission framework to be used with Meteor Accounts.

No sweat if you aren't using these packages, the concepts are still 100% applicable to your application.

```javascript
// a couple helpers that we will use in the utils
// Promisifying the Meteor.apply makes for better organization and more consistent code
apply = Promise.denodeify(Meteor.apply)
// used for creating dynamic it statements (explicit > implicit)
formatNumber = (num) ->
  moment(num, 'D').format('Do')

utils = {};

utils.test = {

  // turn the meteor call into a Promise to create a more predictable environment and
  // so that we can easily handle output. Check out the docs on [Promise.nodeify](https://www.promisejs.org/api/). We use it to call the done function at the end of each test with out having to write out the whole function block, simply cleans up the code a bit.
  promiseCall: () => {
    Promise.nodeify(Meteor.call)
  },

  checkArgument: (method, args, type, setup) => {
    it(`expects a ${type} as the ${formatNumber(args.length)} argument`, (done) => {
      if (setup) { setup() }
      apply(method, args)
        .then( => { throw new Meteor.error('.then should not have been called', 'error!') })
        .catch( (err) => {                  
          expect(err).to.have.property('message').to.equal(`Match error: Expected ${type}, got ${typeof args[args.length - 1]}`)
          expect(err).to.have.property('errorType').to.equal('Match.Error')})
        .nodeify(done)
    });
  },

  checkUserLoggedIn: () => {
    it("throws if the user is not logged in", (done) => {      
      apply(method)
        .then( => { throw new Meteor.error('.then should not have been called', 'error!') })
        .catch((error) => {
          expect(error).to.have.property('errorType').to.equal 'Meteor.Error'
          expect(error).to.have.property('error').to.equal 'not-allowed'
          expect(error).to.have.property('reason').to.equal 'Altoid?'})
        .nodeify(done)
    })
  },

  checkPermission: (method, args, setup) => {
    it("throws if the user does not have the correct permissions", (done) => {
      if (setup) { setup() }
      apply(method, args)
      .then( => { throw new Meteor.error('.then should not have been called', 'error!') })
      .catch((error) => {        
        expect(error).to.have.property('errorType').to.equal('Meteor.Error')
        expect(error).to.have.property('error').to.equal('not-allowed')
        expect(error).to.have.property('reason').to.equal('You don\'t have the right permission for this operation')
      })
      .nodeify(done)
    })
  },

  // we test our validation separately so all we need to do here is ensure that the validate method is called
  checkValidation: (method, args, collection, setup) => {
    it("calls the validate function on #{method}", (done) => {      
      if (setup) { setup() }
      validate = sinon.spy(collection.prototype, 'validate')
      apply(method, args)
        .then( => {
          wasCalled = validate.called
          validate.restore()
          wasCalled.should.equal.true
          done()          
        })
    })
  },
}
```

Now we can do this:

```javascript
if (Meteor.isServer) {
  describe( 'updateDocument function', => {
    // check for error when wrong arg type is passed
    utils.test.checkArgument('updateDocument', [123], String)
    utils.test.checkArgument('updateDocument', ['docId123', 123], String)
    utils.test.checkArgument('updateDocument', ['docId123', 123, []], Object)

    utils.test.checkUserLoggedIn('updateDocument')

    utils.test.checkPermission('updateDocument', ['docId123', 'Posts', {title: 'Test4Life'}])

    utils.test.checkValidation('updateDocument', ['docId123', 'Posts', {title: 'Test4Life'}])

    // ... finally on to the success case
  })

}

```

##### Synchronous:
Wrapping the Meteor.apply call in a resolved Promise does two things:
  1. organizes our code into a more readable set of statements
  1. makes our tests more consistent because we know they will execute synchronously

##### Flexible:
Another cool feature of these utils is that a couple of them take a 'setup' function as an argument. This is really handy. For example, when tackling a special case and you need to stub or spy on a function before the test runs. Just define a function which includes any code that you need to be run before the test executes. When stubbing or spying in this 'setup' function, make sure you use a [sinon sandbox](http://sinonjs.org/docs/#sandbox). That way you clan clean it up in a afterEach simply by calling `sandbox.restore()`. Checkout this [great tutorial](https://semaphoreci.com/community/tutorials/best-practices-for-spies-stubs-and-mocks-in-sinon-js).

##### Explicit:
You'll notice the 'then' blocks that throw an error no matter what. We expect this test to always end up in the catch block because we are testing for an error. Without the then block, this test would fail silently, and actually look like a pass. We try to return as much information back to the developer so that they can get back to fixing the issues after running the tests.  In a later tutorial we will refactor to use the chai-as-promised library.

So if we create a set of utilities to abstract out a bunch of redundant code, then our team is more likely to do things consistently, put all the security checks in place and test for them as well. CRUSH!


### Take aways:

* We defined a file structure that allows clear separation and organization of your code, making it easy for a large team to isolate tests that they are working on. This facilitates rapid iteration and encourages understanding of the test suite.
* By using promises to enforce synchronicity we have created a stable environment for more reliable results. This pattern also contributes greatly to code readability.
* Using a set of test utils standardizes not only test code but source code as well. We have reduced the number of lines needed to write thorough tests, which always helps a team follow through on high quality coverage.

With these concepts and tools, you should be on your way to a rock solid Meteor testing environment. Thanks for reading.
