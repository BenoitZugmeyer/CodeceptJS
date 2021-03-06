# Page Object

UI of your web application has interaction areas which can be shared across different tests.
To avoid code duplication you can put common locators and methods into one place.

## Actor

At initialization you were asked to create custom steps file. If you accepted this option you may use `custom_steps.js` file to extend `I`. See how `login` method can be added to `I`:

```js
module.exports = function() {
  return actor({

    login: function(email, password) {
      this.fillField('Email', email);
      this.fillField('Password', password);
      this.click('Submit');
    }
  });
}
```

Please notice that instead of `I` you should use `this` in current context.

## PageObject

In case an application has different pages (login, admin, etc) you should use a page object.
CodeceptJS can generate a template for it with the command:

```sh
codeceptjs gpo
```

This will create a sample template for a page object and include it into `codecept.json` config.

```js
const I = actor();

module.exports = {

  // insert your locators and methods here
}
```

As you see, `I` object is available there so you can use it as you do in tests.
General page object for a login page may look like this:

```js
const I = actor();

module.exports = {

  // setting locators
  fields: {
    email: '#user_basic_email',
    password: '#user_basic_password'
  },
  submitButton: {css: '#new_user_basic input[type=submit]'},

  // introducing methods
  sendForm(email, password) {
    I.fillField(this.fields.email, email);
    I.fillField(this.fields.password, password);
    I.click(this.submitButton);
  }
}
```

You can include this pageobject in test by its name (defined in `codecept.json`). In case you created a `loginPage` object
it should be added to list of test arguments to be included in test:

```js
Scenario('login', (I, loginPage) => {
  loginPage.sendForm('john@doe.com','123456');
  I.see('Hello, John');
});
```

Also you can use `async/await` inside PageObject:

```js
const I = actor();

module.exports = {

  // setting locators
  container: "//div[@class = 'numbers']",
  mainItem: {
    number: ".//div[contains(@class, 'numbers__main-number')]",
    title: ".//div[contains(@class, 'numbers__main-title-block')]"
  },

  // introducing methods
  openMainArticle: async () => {
    I.waitForVisible(this.container)
    let _this = this
    let title;
    await within(this.container, async () => {
      title = await I.grabTextFrom(_this.mainItem.number);
      let subtitle = await I.grabTextFrom(_this.mainItem.title);
      title = title + " " + subtitle.charAt(0).toLowerCase() + subtitle.slice(1);
      await I.click(_this.mainItem.title)
    })
    return title;
  }
}
```

and use them in your tests:

```js
Scenario('login2', async (I, loginPage, basePage) => {
  let title = await mainPage.openMainArticle()
  basePage.pageShouldBeOpened(title)
});
```

## Page Fragments

In a similar manner CodeceptJS allows you to generate **PageFragments** and any other are abstraction
by running `go` command with `--type` (or `-t`) option:

```sh
codeceptjs go --type fragment
```

Page Fragments represent autonomous parts of a page, like modal boxes, components, widgets.
Technically they are the same as PageObject but conceptually they are a bit different.
For instance, it is recommended that Page Fragment to include a root locator of a component.
Methods of page fragment can use `within` block to narrow scope to a root locator:

```js
const I = actor();
// fragments/modal.js
module.exports = {

  root: '#modal',

  // we are clicking "Accept: inside a popup window
  accept() {
    within(this.root, function() {
      I.click('Accept');
    });
  }
}
```

## StepObjects

StepObjects represent complex actions which involve usage of multiple web pages. For instance, creating users in backend, changing permissions, etc.
StepObject can be created similarly to PageObjects or PageFragments:

```sh
codeceptjs go --type step
```

Technically they are the same as PageObjects. StepObjects can inject PageObjects and use multiple POs to make a complex scenarios:

```js
const I = actor();
const userPage = require('../pages/user');
const permissionPage = require('../pages/permissions');

module.exports = {

  createUser(name) {
    // action composed from actions of page objects
    userPage.open();
    userPage.create(name);
    permissionPage.activate(name);
  }

};
```

## Dependency Injection

### Configuration

All objects described here are injected with Dependency Injection. The similar way it happens in AngularJS framework.
If you want an object to be injected in scenario by its name add it to configuration:

```js
  "include": {
    "I": "./custom_steps.js",
    "Smth": "./pages/Smth.js",
    "loginPage": "./pages/Login.js",
    "signinFragment": "./fragments/Signin.js"
  }
```

Now this objects can be retrieved by the name specified in configuration.
CodeceptJS generator commands (like `codeceptjs gpo`) will update configuration for you.

### Dynamic Injection

You can inject objects per test by calling `injectDependencies` function on Scenario:

```js
Scenario('search @grop', (I, Data) => {
  I.fillField('Username', Data.username);
  I.pressKey('Enter');
}).injectDependencies({ Data: require('./data.js') });
```

This requires `./data.js` module and assigns it to `Data` argument in a test.

### done()
