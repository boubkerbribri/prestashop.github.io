---
layout: post
title:  "Automated UI tests tutorial"
subtitle: "Step by step - Adding a new PrestaShop UI test"
date:  2020-09-25 12:00:00
authors: [boubkerbribri]
icon: icon-github-alt
image: /assets/images/theme/meta-logo-build.png
tags: [community, contribution, tests, automation]
---

One year ago, the QA team has developed a new test framework, based on [puppeteer](https://github.com/puppeteer/puppeteer), [mocha](https://mochajs.org/), and [chai](https://www.chaijs.com/).
Since then, tests coverage has been continually increased, we did also improve our framework by switching to [playwright](https://playwright.dev/) (instead of puppeteer).

In this article, we will show how to create a new user interface test in [PrestaShop core project](https://github.com/PrestaShop/PrestaShop).

Before reading this article, you should read about (and understand) our stack and architecture in [PrestaShop documentation](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/). 


## I. Writing the scenario

The first step is likely the most important one. Why? Because after finishing, you will understand your scenario.

Writing your scenario begins by knowing what your test will do (For this example : Checkin that customer link in orders page in BO redirects to the view customer page).
After that, you perform the test manually, to be sure it's working, and to write (in sentences first) all steps needed.

Now, that you have the scenario, you can create a new javascript file and write your scenario in mocha way (see example below).

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

Note 1: The directory in which you create your file should be chosen wisely (which campaign ? BO or FO ? Which page in BO ? ...).

Note 2: You can have nested describes (scenarios inside scenarios) in the same file (check [mocha documentation](https://mochajs.org/) for more information).

Note 3: We recommend adding more information about the scenario as a comment before your describe (so anyone that open the file see what the test exactly do).

```js
/*
Go to orders page
Filter Bu cutomer name 'J. DOE'
Click on customer link on grid
Check that 'View customer' page is displayed 
 */
```

## II. Opening the browser tab

For each and every UI test, you need a browser, in our framework, we are using a helper to wrap playwright functions that open browser, browser context and tab.

The helper file is located in the utils directory, you can require it using module-alias as below :

```js
require('module-alias/register');

const helper = require('@utils/helpers');
```

Now, that you have the helper, you should be able to create the [`before`](https://mochajs.org/#hooks) function inside your describe.

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

Note : In the before function, we only open the browser context, and the page (= browser tab), we don't worry about the browser because it's handled by the [setup file](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/#setup) (a file executed by mocha before each run).

## III. Using common tests

Some steps are used repeatedly in a lot of scenarios, so we decide to store these steps in a folder called **commonTests**.

Login BO (With demo admin’s account) is part of common tests, and you could use in your new test like below :

```js
it('should login in BO', async function () {
  await loginCommon.loginBO(this, page);
});
```

Note 1: Before using it, you should require the file.

```js
// Require common test login
const loginCommon = require('@commonTests/loginBO');
```

Note 2: You could use another user and password that the demo admin’s account.

```js
it('should login in BO', async function () {
  await loginCommon.loginBO(this, page, 'yourLogin', 'yourPassword');
});
```

## IV. Requiring needed pages

Each test has it needs, after writing the scenario, you knew which pages are needed (for this example, we need BO pages : dashboard, orders, and view customer).

```js
// Import pages
const dashboardPage = require('@pages/BO/dashboard');
const ordersPage = require('@pages/BO/orders');
const viweCustomerPage = require('@pages/BO/customers/view');
```

Note 1: To found the exact locations of page to require, check this [link](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/#pages) on our docs.

Note 2: If a page don’t exist yet, you need to create it (in the right folder).



## V. Filling the steps

Two parts composed each step of the scenario : actions and expected Results. There are functions written or will be written on your pages.
Before adding them, you should check for existing ones and their parameters, or think about parameters for the new ones.

### Actions

The actions are the part when the user do something on the browser : click on link, fill a form ... 

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

Note: For filter orders, we used the FO default account which is part of demo data stored in *data* directory. For this scenario, we require customer file

```js
// Import customer 'J. DOE'
const {DefaultAccount} = require('@data/demo/customer');
```

### Expected results

After each action done on the browser, the user (here the test framework) have to ckeck if the actions is well done (Check a text, a number , an element in the page ...). That's what we call : Expected results.

For that we use *expect* from [Chai library](https://www.chaijs.com/).

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
  await expect(pageTitle).to.contains(ordersPage.pageTitle);
});

it('should reset all filters', async function () {
  // Action
  const numberOfOrders = await ordersPage.resetAndGetNumberOfLines(page);

  // Expected result
  // Check that number of orders > 0 (that there's element in the list) 
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
  // Check that we have at least 1 order with customer J. DOE
  const numberOfOrders = await ordersPage.getNumberOfElementInGrid(page);
  await expect(numberOfOrders).to.be.at.least(1);
});

it('should check customer link', async function () {
  // Action
  page = await ordersPage.viewCustomer(page, 1);

  // Expected result
  // Verify that the current page is view customer by checking the title 'View information about J. DOE'
  const pageTitle = await viweCustomerPage.getPageTitle(page);
  await expect(pageTitle).to
    .contains(`${viweCustomerPage.pageTitle} ${DefaultAccount.firstName[0]}. ${DefaultAccount.lastName}`);
});
```

Note: Before using expect, you should require it.

```js
// Import expect from chai
const {expect} = require('chai');

```
### Implementing missing function

For this scenario, we used mostly existing functions like *filterOrders*. You can check for existing functions in the *pages* directory. 

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

But there is a chance that all functions are not already implemented. In this tutorial, it's the case of *viewCustomer* function.
 
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

After finishing writing your scenario, implement missing functions, it's time to test it and fix (if it fails). 

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

Steps identifiers helps us to track failing steps between different runs in [nightly](https://nightly.prestashop.com/).

Each step should have its own, You can found more about it in this [link](https://devdocs.prestashop.com/1.7/testing/ui-tests/how-to-contribute-and-create-ui-tests/#test-identifier).

After adding it, Each step should look like this example :

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

Eslint is a linter tool for identifying and reporting on patterns in JavaScript. It's used by QA team also, to run it, you can use the command *npm run lint*, you have to fix errors (if there are any).

```shell script
npm run lint

> eslint  --fix --ignore-path .gitignore .

```

## IX. Creating your pull request

Now, that your test is ready, and you want to add it to PrestaShop tests campaigns, you can create a pull request by following the [contribution guidelines](https://github.com/PrestaShop/PrestaShop#contributing).

Link to the Pr for this example : [#20280](https://github.com/PrestaShop/PrestaShop/pull/20280).
