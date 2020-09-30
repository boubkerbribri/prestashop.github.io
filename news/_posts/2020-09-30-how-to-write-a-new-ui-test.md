---
layout: post
title:  "Automated UI tests tutorial"
subtitle: "Step by step - Adding a new PrestaShop UI test"
date:  2020-09-30 12:00:00
authors: [boubkerbribri]
icon: icon-github-alt
image: /assets/images/theme/meta-logo-build.png
tags: [community, contribution, tests, automation]
---

One year ago, the QA team started to develop a new test framework, based on [Puppeteer](https://github.com/puppeteer/puppeteer), [Mocha](https://mochajs.org/), and [Chai](https://www.chaijs.com/).
Since then, the tests coverage has been continually increasing and we recently improved our framework by switching to [Playwright](https://playwright.dev/).

In this article, we will explain how to create a new UI (User Interface) test for the [PrestaShop core project](https://github.com/PrestaShop/PrestaShop).

Before we start, you should definitely read about our stack and architecture in the [PrestaShop devdoc](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/). 


## I. Writing the scenario

Writing your scenario begins by knowing what exactly you need to test and limiting the scope. For this example : **we will be checking that the customer link in the Orders page in the backoffice redirects to the view customer page**.
The second step involves a manual check of this very scenario to be sure it's working, and to write (on a piece of paper) all the steps needed, from start to finish.

Now that you have the scenario, you can create a new javascript file and write your scenario using Mocha (see example below).

```js
// 'describe' = scenario
// 'it' = step

describe('View customer form orders page', async function(){
  it('should login in BO');

  it('should go to orders page');

  it('should reset all filters');

  it('should filter orders by customer name');

  it('should check customer link');

});
```

Note 1: The directory in which you create your file should be chosen wisely (which campaign ? BO or FO ? Which page in BO ? ...). This file organization is very important since Mocha lets you run test by folders (recursively or not), so grouping your tests by features or meta-features (pages) is a very good idea.

Note 2: You can create nested describes (scenarios inside scenarios) in the same file, if you need to regroup some tasks in a more readable way. Keep in mind that the generated report will follow your hierarchy !

Note 3: We recommend adding more information about the scenario as a comment before your describe (so anyone opening the file can see what the test exactly do).

```js
/*
Go to orders page
Filter by customer name 'J. DOE'
Click on customer link on grid
Check that 'View customer' page is displayed 
 */
```

## II. Opening the browser tab

For each and every UI test, you need a browser. The framework the QA team developed includes a set of helpers function to abstract all of this: open a browser, create a new browser context and/or a new tab.

The helper file is located in the `utils` directory and you can require it using the module-alias library as below :

```js
require('module-alias/register');

const helper = require('@utils/helpers');
```

Once you included the helper, you are able to create the [`before`](https://mochajs.org/#hooks) function inside your describe.

```js
const helper = require('@utils/helpers');

let browserContext;
let page;

/*
Go to orders page
Filter Bu cutomer name 'J. DOE'
Click on customer link on grid
Check that 'View customer' page is displayed 
 */
describe('View customer form orders page', async function(){

  before(async function () {
    browserContext = await helper.createBrowserContext(this.browser);
    page = await helper.newTab(browserContext);
  });

  it('should login in BO');

  it('should go to orders page');

  it('should reset all filters');

  it('should filter orders by customer name');

  it('should check customer link');

});
```

Note : In the before function, we only open the browser context and the page (= browser tab) through the `createBrowserContext()` method. We don't worry about opening the browser itself because it's handled by the [setup file](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/#setup), which is a file executed by mocha before each run.

## III. Using common tests

Some steps are used repeatedly in a lot of scenarios, so the QA team decided to store them in a folder called `commonTests`.

First we need to require the file containing the steps we're interessed in, still using the require module like before:

```js
// Require common test login
const loginCommon = require('@commonTests/loginBO');
```
`Login BO` (with the default admin account) is a part of these common tests, and you can use in your new test like this :

```js
it('should login in BO', async function () {
  await loginCommon.loginBO(this, page);
});
```
Note: You can specify another user and password.

```js
it('should login in BO', async function () {
  await loginCommon.loginBO(this, page, 'yourLogin', 'yourPassword');
});
```

## IV. Requiring needed pages

Each test has its needs! After writing the scenario, you know which pages are needed (for this example, we need the following backoffice pages : dashboard, orders, and view customer).

```js
// Import pages
const dashboardPage = require('@pages/BO/dashboard');
const ordersPage = require('@pages/BO/orders');
const viweCustomerPage = require('@pages/BO/customers/view');
```

Note 1: To found the exact locations of the pages to require, check this [link](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/#pages) on our docs, or explore the `pages` folder.

Note 2: If a page doesn’t exist yet, you need to create it (in the right folder).



## V. Filling the steps

Each step of the scenario is made of two parts: **actions** and **expected results** (through data reporting). These are functions in your pages.

Before adding them, you should check for existing ones and their parameters.

### Actions

Actions are replicating what an user would do in the page (clicking on an item, filling an input, etc).

```js
it('should go to orders page', async function () {
  // Action
  // Open Menu and go to orders page
  await dashboardPage.goToSubMenu(
    page, // Browser tab
    dashboardPage.ordersParentLink, // Parent menu item : Sell -> orders
    dashboardPage.ordersLink, // Child menu item : Sell -> orders -> orders
  );
});

it('should reset all filters', async function () {
  // Action
  // Reset filter in orders list and get bumber of element
  await ordersPage.resetAndGetNumberOfLines(
    page, // Browser tab
  );
});

it('should filter order by customer name', async function () {
  // Action
  // Filter order by filling 'customer' input with 'DOE' (lastname of FO default account) 
  await ordersPage.filterOrders(
    page, // Browser tab
    'input', // Filter type (input or select)
    'customer', // Filter column
    DefaultAccount.lastName, // Filter value 'DOE'
  );
});

it('should check customer link', async function () {
  // Action
  // Click on customer link in first row
  await ordersPage.viewCustomer(
    page, // Browser tab
    1, // First row in list
  );
});
```
Here we use differents **actions** methods, like `goToSubMenu()` in the dashboard page, or `resetAndGetNumberOfLines()` in the orders page.

Note: For filter orders, we used the FO default account which is part of demo data stored in *data* directory. For this scenario, you need to require this file:

```js
// Import customer 'J. DOE'
const {DefaultAccount} = require('@data/demo/customer');
```

### Expected results

Data reporting (or "expected results") is done by checking elements in the page and returning a value (number of lines in a table, content of a modal, value of an input, etc).

We use the *expect* element from the [Chai library](https://www.chaijs.com/) and you need to require it:

```js
// Import expect from chai
const {expect} = require('chai');
```

You can then add your `expect` calls in the test logic:

```js
it('should go to orders page', async function () {
  // Action
  await dashboardPage.goToSubMenu(
    page,
    dashboardPage.ordersParentLink,
    dashboardPage.ordersLink,
  );

  // Expected result
  // Verify that the current page is orders by checking the title
  const pageTitle = await ordersPage.getPageTitle(page);
  await expect(pageTitle).to.contains(ordersPage.pageTitle); //we compare with the value stored in the page object
});

it('should reset all filters', async function () {
  // Action
  const numberOfOrders = await ordersPage.resetAndGetNumberOfLines(page);

  // Expected result
  // Check that number of orders > 0 (that there's at least one element in the list) 
  await expect(numberOfOrders).to.be.above(0);
});

it('should filter order by customer name', async function () {
  // Action
  await ordersPage.filterOrders(
    page,
    'input',
    'customer',
    DefaultAccount.lastName,
  );

  // Expected result
  // Check that we have at least 1 order for the customer named "J. DOE"
  const numberOfOrders = await ordersPage.getNumberOfElementInGrid(page);
  await expect(numberOfOrders).to.be.at.least(1);
});

it('should check customer link', async function () {
  // Action
  page = await ordersPage.viewCustomer(page, 1);

  // Expected result
  // Verify that the current page is "view customer" by checking the title 'View information about J. DOE'
  const pageTitle = await viweCustomerPage.getPageTitle(page);
  await expect(pageTitle).to
    .contains(`${viweCustomerPage.pageTitle} ${DefaultAccount.firstName[0]}. ${DefaultAccount.lastName}`);
});
```

### Implementing a missing method

For this scenario, we used mostly existing functions like *filterOrders()*. You can check for existing functions in the `pages` directory. 

```js
  /**
   * Filter Orders
   * @param page
   * @param filterType
   * @param filterBy
   * @param value
   * @return {Promise<void>}
   */
  async filterOrders(page, filterType, filterBy, value = '') {
    switch (filterType) {
      case 'input':
        await this.setValue(page, this.filterColumn(filterBy), value.toString());
        break;

      case 'select':
        await this.selectByVisibleText(page, this.filterColumn(filterBy), value);
        break;

      default:
      throw new Error(`${filterBy} was not found as a column filter.`);
    }
    // click on search
    await this.clickAndWaitForNavigation(page, this.filterSearchButton);
  }
```

But there's a chance that a method you need is not implemented yet. Here it's the case for the *viewCustomer()* function.
 
```js
/**
* Click on customer link to open view page in a new tab
* @param page
* @param row
* @return {Promise<*>}, new browser tab to work with
*/
viewCustomer(page, row) {
  return this.openLinkWithTargetBlank(
    page,
    `${this.tableColumn(row, 'customer')} a`,
    this.userProfileIcon,
  );
}
```

## VI. Running your scenario

After finishing writing your scenario, implementing missing functions... it's time to test your test ! 

```shell script
TEST_PATH="functional/BO/02_orders/01_orders/08_viewCustomer" URL_FO=shopUrl/ npm run specific-test

 View customer from orders page
    ✓ should login in BO
    ✓ should go to orders page
    ✓ should reset all filters
    ✓ should filter order by customer name
    ✓ should check customer link
 5 passing (19s)
```

## VII. Adding steps Identifiers

Steps identifiers help us track failing steps between different runs thanks to the [nightly](https://nightly.prestashop.com/) algorithm.

Each step should have its own ! You can found more information [here](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/#test-identifier).

After adding this identifier, each step should look like this :

```js
it('should check customer link', async function () {
  await testContext.addContextItem(this, 'testIdentifier', 'viewCustomer', baseContext);

  // Click on customer link in first row
  page = await ordersPage.viewCustomer(page, 1);

  // Verify that the current page is view customer by checking the title
  const pageTitle = await viweCustomerPage.getPageTitle(page);
  await expect(pageTitle).to
    .contains(`${viweCustomerPage.pageTitle} ${DefaultAccount.firstName[0]}. ${DefaultAccount.lastName}`);
});
```

Note : No need to add a step identifier to common steps, they already have it.

## VIII. Running eslint

Eslint is a linter tool used to identify and report on patterns in JavaScript. It's currently used by the QA team. To run it, use the command *npm run lint*. you may have to fix errors.

```shell script
npm run lint

> eslint  --fix --ignore-path .gitignore .

```

## IX. Creating your pull request

Now that your test is ready, you want to add it to the PrestaShop tests campaign ! You can create a pull request by following the [contribution guidelines](https://github.com/PrestaShop/PrestaShop#contributing). As usual, thanks a lot for helping this Open Source project.

Link to the PR used for this example : [#20280](https://github.com/PrestaShop/PrestaShop/pull/20280).
