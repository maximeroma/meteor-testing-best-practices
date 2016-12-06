### client or server, pick one

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

### Test Utils help you standardize code

Creating a set of test utilities gives structure to your code before you even write it!

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

Thats a lot of tests to check the standard function and security of an argument, but super important if you ask me. Here are few utils that you can use to make this process more standardized as well as encourage best practices within your team.

```javascript

apply = Promise.denodeify(Meteor.apply)
utils = {};

utils.test = {

  // turn the meteor call into a Promise to create a more predictable environment and
  // so that we can easily handle output
  promiseCall: () => {
    Promise.nodeify(Meteor.call)
  },

  checkArgument: (method, args, type, setup) => {
    it("expects a #{type} as the #{formatNumber args.length} argument", (done) => {
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

Synchronous:
Wrapping the Meteor.apply call in a resolved Promise makes our tests more consistent because we know they will execute synchronously.

Flexible:
Another cool feature of these utils is that a couple of them take a setup function as an argument. This is really handy when tackling a special case and you need to stub or spy on a function.

Explicit:
We try to return as much information back to the developer so that they can get back to fixing the issues after running the tests. You'll notice those catches. They are there incase an error is not thrown, therefore not triggering the catch. In a later tutorial we will refactor to use the chai-as-promised library.


So if we create a set of utilities to abstract out a bunch of redundant code, then our team is more likely to do things consistently, put all these security checks in place and test for them as well. CRUSH!


sandbox, clean up after yourself
synchronous test runner, no finny business
chai as promised
