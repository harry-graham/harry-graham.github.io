---
layout: post
title: "Implementing the Factory Pattern 👨‍🏭"
date: "2023-07-12"
---

## Chapter 1: Acknowledging the Pain Point

At work, we had a new TypeScript API, for which we used Cypress for end-to-end testing.

When the first end-to-end tests were written, the test data was managed as follows:

- A `beforeEach` hook created the required test data, via a POST request to the relevant endpoint
- The test data object's ID was retrieved from the POST request's response body
- The ID was wrapped within a variable, for later access
- The variable was later accessed in each test as needed

This approach was complex, and had many drawbacks, including being:

- Difficult to read (verbose, complex, not DAMP)
- Inefficient/wasteful - all test objects were created for every test, even if they weren't needed
- Time-consuming to set up and maintain
- Required a lot of mental energy to read and understand what was happening each time
- Brittle

### Example

To demonstrate this, here is an example using "list" objects, where each list has several "items":

```typescript
describe("List Items", () => {
  // Shared Setup: Create list object
  beforeEach(() => {
    cy.authorisedRequest(
      method: "POST",
      url: "/lists",
      body: {
        name: "Shopping List",
      },
    ).then((resp) => {
      cy.wrap(resp.body["id"]).as("listId");
    });
  });

  it("Creates list item successfully", () => {
    cy.get("@listId").then((listId) => {
      cy.authorisedRequest({
        method: "POST",
        url: `/lists/${listId}/items`,
        body: {
          name: "Apples",
        },
      }).then((resp) => {
        expect(resp.status).to.eq(201);
        expect(resp.headers.location).to.eq(`/lists/${listId}/items`);
      });

      cy.authorisedRequest({
        method: "GET",
        url: `/lists/${listId}/items`,
      }).then((resp) => {
        expect(resp.status).to.eq(200);
        expect(resp.body.length).to.eq(1);
        expect(resp.body[0]).to.have.property("listId", listId);
        expect(resp.body[0]).to.have.property("name", "Apples");
      });
    });
  });

  it("Returns 400 if name is not unique", () => {
    cy.get("@listId").then((listId) => {
      cy.authorisedRequest({
        method: "POST",
        url: `/lists/${listId}/items`,
        body: {
          name: "Apples",
        },
      }).then((resp) => {
        expect(resp.status).to.eq(201);
        expect(resp.headers.location).to.eq(`/lists/${listId}/items`);
      });

      cy.authorisedRequest({
        method: "POST",
        url: `/lists/${listId}/items`,
        body: {
          name: "Apples",
        },
        failOnStatusCode: false,
      }).then((resp) => {
        expect(resp.status).to.eq(400);
        expect(resp.body).to.eq("Name must be unique");
      });
    });
  });
});
```

Even with this simplified example, this is quite complex and confusing, and takes some navigating and re-reading at first to understand what is happening.

## Chapter 2: Implementing the Factory Pattern 🤖

### Why?

Using the factory pattern allows for creating new test objects within tests in an incredibly simple and readable fashion.

### How?

In Ruby, the factory pattern is frequently seen and used, often using the `factory-bot` Ruby gem.

TypeScript doesn't have this code package, but it is still possible to create factories manually for each required test object, to gain the benefits of the factory pattern, as I have done here.

One extra thing to note: `faker` can be used to auto-generate test data values, as I have done below. It is available for many languages, including both Ruby and TypeScript.

### Example

Here is an example test factory for `list` objects, defined in a separate file.

```typescript
import { faker } from "@faker-js/faker";
...

function defaultList(): List {
  return {
    id: faker.datatype.uuid(),
    name: "Default List Name",
  };
}

function createListInDB(list: List) {
  cy.authorisedRequest(
    method: "POST",
    url: "/lists",
    body: { ...list },
  );
};

export const createList = (overwrites: Partial<List> = {}): List => {
  const list = { ...defaultList(), ...overwrites };

  createListInDB(list);

  return list;
};
```

The factory defines 1 exported method, `createList`, which:

- Optionally takes an object (to override the values of any properties of the test object, as desired)
- Assigns default values to any properties that aren't overriden (if defined in the factory)
- Returns the new test object

### Review of this pattern

There may be a level of complexity here, but the code feels clean and is abstracted away from the end-to-end tests into its own file. As a result, the tests are simpler and easier to read.

Overall, this approach is reusable, DRY, scaleable, and easier to maintain, whilst still allowing for DAMP and readable tests.

## Chapter 3: The Result

The factory made the existing tests simpler, more DAMP, and more readable:

```typescript
describe("List Items", () => {
  it("Creates list item successfully", () => {
    const list = createList({
      name: "Shopping List",
    });

    cy.authorisedRequest({
      method: "POST",
      url: `/lists/${list.id}/items`,
      body: {
        name: "Apples",
      },
    }).then((resp) => {
      expect(resp.status).to.eq(201);
      expect(resp.headers.location).to.eq(`/lists/${list.id}/items`);
    });

    cy.authorisedRequest({
      method: "GET",
      url: `/lists/${list.id}/items`,
    }).then((resp) => {
      expect(resp.status).to.eq(200);
      expect(resp.body.length).to.eq(1);
      expect(resp.body[0]).to.have.property("listId", list.id);
      expect(resp.body[0]).to.have.property("name", "Apples");
    });
  });

  it("Returns 400 if name is not unique", () => {
    const list = createList({
      name: "Shopping List",
    });

    cy.authorisedRequest({
      method: "POST",
      url: `/lists/${list.id}/items`,
      body: {
        name: "Apples",
      },
    }).then((resp) => {
      expect(resp.status).to.eq(201);
      expect(resp.headers.location).to.eq(`/lists/${list.id}/items`);
    });

    cy.authorisedRequest({
      method: "POST",
      url: `/lists/${list.id}/items`,
      body: {
        name: "Apples",
      },
      failOnStatusCode: false,
    }).then((resp) => {
      expect(resp.status).to.eq(400);
      expect(resp.body).to.eq("Name must be unique");
    });
  });
});
```
